---
date: 2024-09-16
categories:
  - Python
  - Qt
  - Testing 
  - Ci
tags:
    - Python
    - Testing
    - Article
---

# Preventing segfaults in test suite that has Qt Tests

## Motivation

When providing an GUI application one needs to select GUI backend. 
If application is Python and needs to work on all popular OSes[^1]
the good choice is to use Qt. It is a cross-platform GUI toolkit that
has good python bindings[^2].

However, the Qt objects require special care during testing. 
It this post I will describe my experience of writing such tests based on 
my work on [PartSeg](https://partseg.github.io/) 
and [napari](https://napari.org/).

<!-- more -->

## The problem

As the Qt is C++ library it does not know about python memory management.
This mean that if Qt keep reference to some widget it does not increase reference
count of python object. This can lead to situation when python object is deleted,
but there is still event pending in Qt event loop that reference this object.

When this happens Qt will try to access deleted object, leading to access to unallocated memory (segfault).
This is very hard to debug because segfault can occur in any subsequent
test, making it unclear what the cause is.

The error messages vary across different operating systems:

1. Windows `Windows fatal exception: access violation` 
2. Linux `Segmentation fault (core dumped)` or `Fatal Python error: Aborted` 
3. macOS (I do not have recent runs to check)

Moreover, this behavior is non-deterministic and may be not reproducible locally. 
One of observed source of difference is that the CI runs on server version of OS. 
I have encountered cases when I cannot reproduce the error on my development machine, 
but I could do this on my server. 

### What is segfault

A segfault occurs when a program tries to access memory that is not allocated to it.
In such situations, the OS will kill the program to prevent corruption. 
For security reasons, the OS does not allow handling this error, as it may be caused by malicious code.

An even worse scenario is when the addressed memory is allocated for a different object than the original pointer[^3] was pointing to.
This can lead to modifying a different object in an unpredictable way, causing the test or program to fail unexpectedly.

## How to prevent it

This section is based on my experience and may not be complete.

### Ensure that all qt widgets are scheduled for deletion

All Qt objects have a [`deleteLater`](https://doc.qt.io/qt-6/qobject.html#deleteLater) method that schedules the object for deletion. 
This allows for the safe deletion of the object, ensuring that all pending events are processed.

If you use some widget in test, that is not child of any other widget, 
you should call `deleteLater` on it before the test end.
It is also good practice to ensure that the widget is hidden before deletion. 
So if your test requires showing the widget (e.g. screenshot test) you should hide it before deletion.

When using `pytest` for testing I suggest using the `pytest-qt` plugin.
This plugin provides `qtbot` fixture that can be used to interact with Qt objects.
It also provides a [`qtbot.add_widget`](https://pytest-qt.readthedocs.io/en/latest/reference.html#pytestqt.qtbot.QtBot.addWidget) method that ensures `deleteLater` is called on the widget when the test ends.

If your widget requires special teardown you can use `before_close_func` argument of `add_widget` method.

### Ensure all timers, animations or threads are stopped

I have observed that not stopping [`QTimer`](https://doc.qt.io/qt-6/qtimer.html), 
[`QPropertyAnimation`](https://doc.qt.io/qt-6/qpropertyanimation.html), [`QThread`](https://doc.qt.io/qt-6/qthread.html) or [`QThreadPool`](https://doc.qt.io/qt-6/qthreadpool.html) can lead to segfault.
It may also lead to some other problems with test. 

So if you use any of this objects in test you should ensure that they are stopped before test ends.


### Use the smallest possible widgets for tests

The process of setup and teardown of complex widgets is complex, time-consuming and may contain bugs that are hard to detect.
So if the test purpose is to check behavior of some widget it is better to only create this widget, not the whole window that contains it.


## How to debug and prevent

In this section I will describe my tricks used to debug and prevent segfaults. 
However, it may not fit to all projects.

### Run test under `gdb` or `lldb`

If you could reproduce the segfault locally you can run the test under `gdb` or `lldb`.
Then you could go through the stack trace and see what is the cause of segfault.

There is also option to increase interpolation between `gdb` and python [https://docs.python.org/3/howto/gdb_helpers.html](https://docs.python.org/3/howto/gdb_helpers.html).

You may also build qt in debug mode and compile your python wrapper against it. It will provide more information in stack trace, but is complex and time-consuming.


### Prevent `QThread` and `QTimer` from running

Commonly, the test do not need to use threads. However, it may happen that integration test may trigger some thread. 
It may be a good idea to fail the test if there is call of `QThread.start` method. I use following pytest fixture to do this:

```python
@pytest.fixture(autouse=True)
def _block_threads(monkeypatch, request):
    if "enablethread" in request.keywords:
        return

    from pytestqt.qt_compat import qt_api
    from qtpy.QtCore import QThread, QTimer

    old_start = QTimer.start

    class OldTimer(QTimer):
        def start(self, time=None):
            if time is not None:
                old_start(self, time)
            else:
                old_start(self)

    def not_start(self):
        raise RuntimeError("Thread should not be used in test")

    monkeypatch.setattr(QTimer, "start", not_start)
    monkeypatch.setattr(QThread, "start", not_start)
    monkeypatch.setattr(qt_api.QtCore, "QTimer", OldTimer)
```

As you may see, there is option to allow thread usage by using custom `enablethread` marker. 
The documentation for declaring custom markers is available in [pytest documentation](https://docs.pytest.org/en/stable/example/markers.html#registering-markers).

As documentation do not provide example for `pyproject.toml` I will provide example how to do this:
```toml
[tool.pytest.ini_options]
markers = [
    "enablethread: Allow to use thread in test",
    ...
]
```

You may also spot `monkeypatch.setattr(qt_api.QtCore, "QTimer", OldTimer)` line. It is added because `QTimer` is used internally in `pytest-qt` plugin for `qtbot.wait*` methods.

In similar way you can block usage of `QPropertyAnimation`.

This approach raises exception when non-allowed method is called, so it is easy to prevent unwanted usage of threads.
However it may increase hardness of contributing to project, as it is custom behavior, that potential contributor may not expect.


### Find active timers after test end

In napari project we have developed a `pytest` fixture that checks if there are any active `QTimers`, `QThreads`, `QThreadPool` and `QPropertyAnimation` after test end.

This method is not perfect as it may not be triggered at every test suite run. So problematic code may be detected only after a long time.

```python
@pytest.fixture(auto_use=True)
def dangling_qthreads(monkeypatch, qtbot, request):
    from qtpy.QtCore import QThread

    base_start = QThread.start
    thread_dict = WeakKeyDictionary()

    def start_with_save_reference(self, priority=QThread.InheritPriority):
        """Thread start function with logs to detect hanging threads.

        Saves a weak reference to the thread and detects hanging threads,
        as well as where the threads were started.
        """
        thread_dict[self] = _get_calling_place()
        base_start(self, priority)

    monkeypatch.setattr(QThread, 'start', start_with_save_reference)

    yield

    dangling_threads_li = []

    for thread, calling in thread_dict.items():
        try:
            if thread.isRunning():
                dangling_threads_li.append((thread, calling))
        except RuntimeError as e:
            if (
                'wrapped C/C++ object of type' not in e.args[0]
                and 'Internal C++ object' not in e.args[0]
            ):
                # object was deleted
                raise

    for thread, _ in dangling_threads_li:
        with suppress(RuntimeError):
            thread.quit()
            qtbot.waitUntil(thread.isFinished, timeout=2000)

    long_desc = (
        'If you see this error, it means that a QThread was started in a test '
        'but not terminated. This can cause segfaults in the test suite. '
        'Please use the `qtbot` fixture to wait for the thread to finish. '
    )

    if len(dangling_threads_li) > 1:
        long_desc += ' The QThreads were started in:\n'
    else:
        long_desc += ' The QThread was started in:\n'

    assert not dangling_threads_li, long_desc + '\n'.join(
        x[1] for x in dangling_threads_li
    )
```

It is simplified version of napari fixture. 
You may see full versions in [napari conftest](https://github.com/napari/napari/blob/15c2d7d5ae7c607e3436800328527bd62c421896/napari/conftest.py#L444)

For other problematic objects you can use similar approach. There are proper fixtures in same [`conftest.py`](https://github.com/napari/napari/blob/15c2d7d5ae7c607e3436800328527bd62c421896/napari/conftest.py) file.


### Detect leaked widgets
 
TBA https://github.com/napari/napari/pull/7251


## Bonus tip

Your tests are hanging, but any above solution did not help. What to do?

One of the possible reason is that your code is created some nested event loop by opening [`QDialog`](https://doc.qt.io/qt-6/qdialog.html) 
or [`QMessageBox`](https://doc.qt.io/qt-6/qmessagebox.html) using `exec` method. 
To get error message instead of hanging test I use following pytest fixture:

```python
import pytest

@pytest.fixture(autouse=True)
def _block_message_box(monkeypatch, request):
    def raise_on_call(*_, **__):
        raise RuntimeError("exec_ call")  # pragma: no cover

    monkeypatch.setattr(QMessageBox, "exec_", raise_on_call)
    monkeypatch.setattr(QMessageBox, "exec", raise_on_call)
    monkeypatch.setattr(QMessageBox, "critical", raise_on_call)
    monkeypatch.setattr(QMessageBox, "information", raise_on_call)
    monkeypatch.setattr(QMessageBox, "question", raise_on_call)
    monkeypatch.setattr(QMessageBox, "warning", raise_on_call)
    monkeypatch.setattr("PartSeg.common_gui.error_report.QMessageFromException.exec_", raise_on_call)
    monkeypatch.setattr(QInputDialog, "getText", raise_on_call)
    if "enabledialog" not in request.keywords:
        monkeypatch.setattr(QDialog, "exec_", raise_on_call)
        monkeypatch.setattr(QDialog, "exec", raise_on_call)

```

As you can see I block multiple methods that can create nested event loop.
In some test I need to allow calling `exec` method of `QDialog`,
so I have defined `enabledialog` marker that I can use to allow this call.


```python

@pytest.mark.enabledialog
def test_recent(self, tmp_path, qtbot, monkeypatch):
  ...
```






[^1]: This includes Windows, macOS, and various distributions of Linux.
[^2]: PyQt5, PySide2 for Qt5, PyQT6, PySide6 for Qt6.
[^3]: [https://en.wikipedia.org/wiki/Pointer_(computer_programming)](https://en.wikipedia.org/wiki/Pointer_(computer_programming))
