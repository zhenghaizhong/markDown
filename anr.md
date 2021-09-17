# anr

##1.Input event dispatching timed out
```
Input event dispatching timed out sending to application ActivityRecord{f92b3b u0 com.android.settings/.Settings$DisplaySettingsActivity.  Reason: ActivityRecord{f92b3b u0 com.android.settings/.Settings$DisplaySettingsActivity t38} does not have a focused window
```

####日志
dumpsys/dumpsys_window-a文件下：
```
Last ANR continued
WINDOW MANAGER DISPLAY CONTENTS (dumpsys window displays)
  mLastOrientationSource=ActivityRecord{f92b3b u0 com.android.settings/.Settings$DisplaySettingsActivity t38}
  deepestLastOrientationSource=ActivityRecord{f92b3b u0 com.android.settings/.Settings$DisplaySettingsActivity t38}
  Display: mDisplayId=0 stacks=12
    init=1080x2400 480dpi base=1080x2400 540dpi cur=1080x2400 app=1080x2302 rng=1080x979-2302x2299
    deferred=false mLayoutNeeded=false mTouchExcludeRegion=SkRegion((0,0,1080,2400))

  mLayoutSeq=4318
  mCurrentFocus=null
  mFocusedApp=ActivityRecord{f92b3b u0 com.android.settings/.Settings$DisplaySettingsActivity t38}
```

dumpsys/dumpsys_input文件下：
```
Input Dispatcher State at time of last ANR:
  ANR:
    Time: 2021-08-14 15:17:33
    Reason: ActivityRecord{f92b3b u0 com.android.settings/.Settings$DisplaySettingsActivity t38} does not have a focused window
    Window: ActivityRecord{f92b3b u0 com.android.settings/.Settings$DisplaySettingsActivity t38}
  DispatchEnabled: true
  DispatchFrozen: false
  InputFilterEnabled: false
  FocusedDisplayId: 0
  FocusedApplications:
    displayId=0, name='ActivityRecord{f92b3b u0 com.android.settings/.Settings$DisplaySettingsActivity t38}', dispatchingTimeout=5000ms
  FocusedWindows: <none>
```
main.log文件下：
```
InputDispatcher: Waiting because no window has focus but ActivityRecord{5f0889d u0 ***********} may eventually add a window when it finishes starting up. Will wait for 5000ms

InputDispatcher: Still no focused window. Will drop the event in ****ms
```
####如果有focused app，但是一直没有 focused window，超时就会触发对应应用的anr

InputDispatcher.cpp 机制代码：
```
nsecs_t InputDispatcher::processAnrsLocked() {
    const nsecs_t currentTime = now();
    nsecs_t nextAnrCheck = LONG_LONG_MAX;
    // Check if we are waiting for a focused window to appear. Raise ANR if waited too long
    if (mNoFocusedWindowTimeoutTime.has_value() && mAwaitedFocusedApplication != nullptr) {
        if (currentTime >= *mNoFocusedWindowTimeoutTime) {
            onAnrLocked(mAwaitedFocusedApplication);
            mAwaitedFocusedApplication.clear();
            return LONG_LONG_MIN;
        } else {
            // Keep waiting
            const nsecs_t millisRemaining = ns2ms(*mNoFocusedWindowTimeoutTime - currentTime);
            ALOGW("Still no focused window. Will drop the event in %" PRId64 "ms", millisRemaining);
            nextAnrCheck = *mNoFocusedWindowTimeoutTime;
        }
    }
```
