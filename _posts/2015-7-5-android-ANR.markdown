---
layout: post
title:  "Android ANR(Application Not Responding)"
date:   2015-7-5
---

<p class="intro"><span class="dropcap">A</span>NR的中文意思是应用程序未响应，在android中，如果程序：</p>

 1. 主线程(UI线程)5秒钟没有响应输入操作
 2. BroadcastReceiver 没有在10秒内完成返回

就会导致ANR。因此我们应当避免如下操作：

- 在主线程内进行网络操作
- 在主线程内进行一些缓慢的磁盘操作（例如执行没有优化过的SQL查询）

##SOLUTION:
***
###使用异步消息处理handler
1. 我们首先需要一个handler对象并实例化它；
2. new一个线程来进行耗时操作
3. 获取数据后调用handler.sendMessage(msg)方法来传递消息
4. 在实例化的handler中有个handleMessage方法，在其中可以得到传来的Message对象，我们就可以在这里进行UI更新操作

```java

	public static final int UPDATE_TEXT = 1;
	private Handler handler = new Handler() {
		public void handleMessage(Message msg) {
			switch (msg.what) {
			case UPDATE_TEXT:
				text.setText("Nice to meet you");
				break;
			default:
				break;
			}
		}

	};

	new Thread(new Runnable() {
				@Override
				public void run() {
					Message message = new Message();
					message.what = UPDATE_TEXT;
					handler.sendMessage(message);
				}
			}).start();
```

***
###使用异步任务AsyncTask
AsyncTask背后的实现原理也是基于异步消息处理机制的，只是Android帮我们做了很好的封装而已。
AsyncTask<Params,Progress,Result>内有三个参数：


- Params 启动任务执行的输入参数，比如HTTP请求的URL。
- Progress 后台任务执行的百分比。
- Result 后台执行任务最终返回的结果，比如String,Bitmap。 

使用过AsyncTask最少要重写以下这两个方法：

- doInBackground(Params…) 后台执行，比较耗时的操作都可以放在这里。注意这里不能直接操作UI。此方法在后台线程执行，完成任务的主要工作，通常需要较长的时间。在执行过程中可以调用publicProgress(Progress…)来更新任务的进度。
- onPostExecute(Result)  相当于Handler 处理UI的方式，在这里面可以使用在doInBackground 得到的结果处理操作UI。 此方法在主线程执行，任务执行的结果作为此方法的参数返回

```java
	public class ProgressBarAsyncTask extends AsyncTask<Integer, Integer, String> {  
  
    private TextView textView;  
    private ProgressBar progressBar;  
      
      
    public ProgressBarAsyncTask(TextView textView, ProgressBar progressBar) {  
        super();  
        this.textView = textView;  
        this.progressBar = progressBar;  
    }  
  
  
    /**  
     * 这里的Integer参数对应AsyncTask中的第一个参数   
     * 这里的String返回值对应AsyncTask的第三个参数  
     * 该方法并不运行在UI线程当中，主要用于异步操作，所有在该方法中不能对UI当中的空间进行设置和修改  
     * 但是可以调用publishProgress方法触发onProgressUpdate对UI进行操作  
     */  
    @Override  
    protected String doInBackground(Integer... params) {  
        NetOperator netOperator = new NetOperator();  
        int i = 0;  
        for (i = 10; i <= 100; i+=10) {  
            netOperator.operator();  
            publishProgress(i);  
        }  
        return i + params[0].intValue() + "";  
    }  
  
  
    /**  
     * 这里的String参数对应AsyncTask中的第三个参数（也就是接收doInBackground的返回值）  
     * 在doInBackground方法执行结束之后在运行，并且运行在UI线程当中 可以对UI空间进行设置  
     */  
    @Override  
    protected void onPostExecute(String result) {  
        textView.setText("异步操作执行结束" + result);  
    }  
  
  
    //该方法运行在UI线程当中,并且运行在UI线程当中 可以对UI空间进行设置  
    @Override  
    protected void onPreExecute() {  
        textView.setText("开始执行异步线程");  
    }  
  
  
    /**  
     * 这里的Intege参数对应AsyncTask中的第二个参数  
     * 在doInBackground方法当中，，每次调用publishProgress方法都会触发onProgressUpdate执行  
     * onProgressUpdate是在UI线程中执行，所有可以对UI空间进行操作  
     */  
    @Override  
    protected void onProgressUpdate(Integer... values) {  
        int vlaue = values[0];  
        progressBar.setProgress(vlaue);  
    }  
  }
```
***
到此，ANR的了解及处理也就差不多了，当然处理的方法还有其它的，你可以自己去了解一下。