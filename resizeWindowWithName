import ctypes
from ctypes import wintypes

PARTIAL_TITLE = "WINDOW TITLE"
TARGET_WIDTH = pixels
TARGET_HEIGHT = pixels

user32 = ctypes.WinDLL("user32", use_last_error=True)

SW_RESTORE = 9
SWP_NOZORDER = 0x0004
SWP_SHOWWINDOW = 0x0040
SWP_NOMOVE = 0x0002
SWP_NOSIZE = 0x0001

MONITOR_DEFAULTTONEAREST = 2


class RECT(ctypes.Structure):
    _fields_ = [
        ("left", wintypes.LONG),
        ("top", wintypes.LONG),
        ("right", wintypes.LONG),
        ("bottom", wintypes.LONG),
    ]


class MONITORINFO(ctypes.Structure):
    _fields_ = [
        ("cbSize", wintypes.DWORD),
        ("rcMonitor", RECT),
        ("rcWork", RECT),
        ("dwFlags", wintypes.DWORD),
    ]


def find_window_by_partial_title(partial_title: str):
    matches = []
    partial_title = partial_title.lower()

    EnumWindowsProc = ctypes.WINFUNCTYPE(
        wintypes.BOOL, wintypes.HWND, wintypes.LPARAM
    )

    def callback(hwnd, lparam):
        if not user32.IsWindowVisible(hwnd):
            return True

        length = user32.GetWindowTextLengthW(hwnd)
        if length == 0:
            return True

        buffer = ctypes.create_unicode_buffer(length + 1)
        user32.GetWindowTextW(hwnd, buffer, length + 1)
        title = buffer.value.strip()

        if partial_title in title.lower():
            matches.append((hwnd, title))

        return True

    user32.EnumWindows(EnumWindowsProc(callback), 0)
    return matches[0] if matches else (None, None)


def get_window_rect(hwnd):
    rect = RECT()
    if not user32.GetWindowRect(hwnd, ctypes.byref(rect)):
        err = ctypes.get_last_error()
        raise OSError(f"GetWindowRect failed. Windows error code: {err}")
    return rect


def get_monitor_rect_for_window(hwnd):
    monitor = user32.MonitorFromWindow(hwnd, MONITOR_DEFAULTTONEAREST)
    if not monitor:
        err = ctypes.get_last_error()
        raise OSError(f"MonitorFromWindow failed. Windows error code: {err}")

    mi = MONITORINFO()
    mi.cbSize = ctypes.sizeof(MONITORINFO)

    if not user32.GetMonitorInfoW(monitor, ctypes.byref(mi)):
        err = ctypes.get_last_error()
        raise OSError(f"GetMonitorInfoW failed. Windows error code: {err}")

    return mi.rcMonitor


def main():
    # Better DPI handling
    try:
        ctypes.windll.shcore.SetProcessDpiAwareness(2)  # Per-monitor DPI aware
    except Exception:
        try:
            user32.SetProcessDPIAware()
        except Exception:
            pass

    hwnd, title = find_window_by_partial_title(PARTIAL_TITLE)

    if not hwnd:
        print(f'No visible window found containing "{PARTIAL_TITLE}"')
        return

    print(f'Found window: "{title}"')

    user32.ShowWindow(hwnd, SW_RESTORE)

    # Resize first
    ok = user32.SetWindowPos(
        hwnd,
        None,
        0,
        0,
        TARGET_WIDTH,
        TARGET_HEIGHT,
        SWP_NOZORDER | SWP_SHOWWINDOW | SWP_NOMOVE,
    )
    if not ok:
        err = ctypes.get_last_error()
        print(f"Initial resize failed. Windows error code: {err}")
        return

    # Measure the real outer window size after resize
    rect = get_window_rect(hwnd)
    actual_width = rect.right - rect.left
    actual_height = rect.bottom - rect.top

    # Get the actual monitor the window is on
    monitor_rect = get_monitor_rect_for_window(hwnd)
    monitor_left = monitor_rect.left
    monitor_top = monitor_rect.top
    monitor_width = monitor_rect.right - monitor_rect.left
    monitor_height = monitor_rect.bottom - monitor_rect.top

    # Center horizontally on THAT monitor, pin to top of THAT monitor
    x = monitor_left + (monitor_width - actual_width) // 2
    y = monitor_top

    ok = user32.SetWindowPos(
        hwnd,
        None,
        x,
        y,
        0,
        0,
        SWP_NOZORDER | SWP_SHOWWINDOW | SWP_NOSIZE,
    )
    if not ok:
        err = ctypes.get_last_error()
        print(f"Final move failed. Windows error code: {err}")
        return

    try:
        user32.SetForegroundWindow(hwnd)
    except Exception:
        pass

    print(f"Monitor size: {monitor_width}x{monitor_height}")
    print(f"Actual window size: {actual_width}x{actual_height}")
    print(f'Positioned "{title}" at ({x}, {y})')


if __name__ == "__main__":
    main()
