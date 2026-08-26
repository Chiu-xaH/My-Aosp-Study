# Window 与 View

从上一篇幅[startActivty（2）——启动首个Activity](/startActivty（2）——启动首个Activity.md)的Activity即将完成Resume流程开始，还记得makeVisible吗，最后调用WindowManager的addView传入DectorView并设置他为VISIBLE。

```kotlin
void makeVisible() {
        if (!mWindowAdded) {
            ViewManager wm = getWindowManager();
            // 本次分析的入口点
            wm.addView(mDecor, getWindow().getAttributes());
            mWindowAdded = true;
        }
        mDecor.setVisibility(View.VISIBLE);
    }

```

<div data-block-type="resource" data-resource-type="attachment" data-blob-store-key="docs_enclosure_14639475-c7ed-4091-bc78-80bc5791932f_drag-upload-1787578124214-0" data-file-name="Untitled.mdj" data-file-size="28188"></div>