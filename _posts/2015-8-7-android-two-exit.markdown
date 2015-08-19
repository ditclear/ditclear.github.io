---
layout: post
title:  "Android 实现两次返回退出"
date:   2015-8-7
---

<p class="intro"><span class="dropcap">A</span>ndroid应用中两次返回退出是很常见的，而且也很容易提醒用户，避免误操作。而实现起来也很简单，只需要在你的mainActivity中加入一段代码就可以了。</p>

##CODE

```java

	/**
     * 点击返回键退出程序
     */
    private static Boolean isExit = false;
    private Handler mHandler = new Handler();
    @Override
    public boolean onKeyDown(int keyCode, KeyEvent event) {
        if (keyCode == KeyEvent.KEYCODE_BACK) {
            if (isExit == false) {
                isExit = true;
                Toast.makeText(this, "再按一次退出程序", Toast.LENGTH_SHORT).show();
                mHandler.postDelayed(new Runnable() {
                    @Override
                    public void run() {
                        isExit = false;
                    }
                }, 2000);
            } else {
                finish();
                System.exit(0);
            }
        }
        return false;
    }
```
