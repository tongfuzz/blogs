###关于WebView  <font color=#ff0000>&lt;Input&gt;</font>  标签点击没有反应的解决方法

在与前端调试页面的时候，发现前端的  &lt;input&gt; 标签在ios手机上使用正常，在Android手机上却没有任何反应，还需要我们移动端进行特殊处理，具体处理方式如下，在点击 &lt;input&gt;标签时，我们的WebChromeClient对象中的响应的方法会被回调，具体如下

```java
public class OpenFileWebChromeClient extends WebChromeClient {
		public String TAG = "OpenFileWebChromeClient";
		public static final int REQUEST_FILE_PICKER = 1;
		public ValueCallback<Uri> mFilePathCallback;
		public ValueCallback<Uri[]> mFilePathCallbacks;
		private Activity mContext;
		private TextView textView;

		public OpenFileWebChromeClient(Activity mContext) {
			super();
			this.mContext = mContext;
		}

		/**
		 * Android < 3.0 调用这个方法
		 */
		public void openFileChooser(ValueCallback<Uri> filePathCallback) {
			mFilePathCallback = filePathCallback;
				Intent intent = new Intent(Intent.ACTION_GET_CONTENT);
				intent.addCategory(Intent.CATEGORY_OPENABLE);
				intent.setType("image/*");
				mContext.startActivityForResult(Intent.createChooser(intent, "File Chooser"),
						REQUEST_FILE_PICKER);


		}

		/**
		 * 3.0 + 调用这个方法
		 */
		public void openFileChooser(ValueCallback filePathCallback,
									String acceptType) {
			mFilePathCallback = filePathCallback;
				Intent intent = new Intent(Intent.ACTION_GET_CONTENT);
				intent.addCategory(Intent.CATEGORY_OPENABLE);
				intent.setType("image/*");
				mContext.startActivityForResult(Intent.createChooser(intent, "File Chooser"),
						REQUEST_FILE_PICKER);


		}

		/**
		 * Android > 4.1.1 调用这个方法
		 */
		@Deprecated
		public void openFileChooser(ValueCallback<Uri> filePathCallback,
									String acceptType, String capture) {
			mFilePathCallback = filePathCallback;
				Intent intent = new Intent(Intent.ACTION_GET_CONTENT);
				intent.addCategory(Intent.CATEGORY_OPENABLE);
				intent.setType("image/*");
				LogUtils.e("1111");
				mContext.startActivityForResult(Intent.createChooser(intent, "File Chooser"),
						REQUEST_FILE_PICKER);

		}


		/**
		 * Android >= 5.0 调用这个方法
		 */
		@Override
		public boolean onShowFileChooser(WebView webView,
										 ValueCallback<Uri[]> filePathCallback,
										 WebChromeClient.FileChooserParams fileChooserParams) {
			mFilePathCallbacks = filePathCallback;
				Intent intent = new Intent(Intent.ACTION_GET_CONTENT);
				intent.addCategory(Intent.CATEGORY_OPENABLE);
				intent.setType("image/*");
				LogUtils.e("1111");
				mContext.startActivityForResult(Intent.createChooser(intent, "File Chooser"),
						REQUEST_FILE_PICKER);

			return true;
		}

		@Override
		public void onProgressChanged(WebView webView, int i) {
			super.onProgressChanged(webView, i);
		}
	}
```

我们创建一个OpenFileWebChromeClient类继承WebChromeClient,并重写其中的方法，注意，不同android版本调用的方法不同，具体如上

然后我们调用mWebView.setWebChromeClient(）方法，将其设置到webview中

```java
private OpenFileWebChromeClient mOpenFileWebChromeClient = new OpenFileWebChromeClient(
			this);
mWebView.setWebChromeClient(mOpenFileWebChromeClient);
```


最后我们要在onActivityResult方法中，获取到我们选择的图片的uri，并且通过调用调用onReceiveValue方法将其设置给webview

```java
@Override
protected void onActivityResult(int requestCode, int resultCode, Intent data) {
	super.onActivityResult(requestCode, resultCode, data);
	if (requestCode == OpenFileWebChromeClient.REQUEST_FILE_PICKER) {

			if (mOpenFileWebChromeClient.mFilePathCallback != null) {
				Uri result = data == null || resultCode != Activity.RESULT_OK ? null
						: data.getData();
					mOpenFileWebChromeClient.mFilePathCallback
							.onReceiveValue(result);
				} else {
					mOpenFileWebChromeClient.mFilePathCallback
							.onReceiveValue(null);
				}

			}

			if (mOpenFileWebChromeClient.mFilePathCallbacks != null) {
				Uri result = data == null || resultCode != Activity.RESULT_OK ? null
						: data.getData();
				Uri result = data == null || resultCode != Activity.RESULT_OK ? null
						: data.getData();
					mOpenFileWebChromeClient.mFilePathCallback
							.onReceiveValue(result);
				} else {
					mOpenFileWebChromeClient.mFilePathCallback
							.onReceiveValue(null);
				}
			}
			mOpenFileWebChromeClient.mFilePathCallback = null;
			mOpenFileWebChromeClient.mFilePathCallbacks = null;
		}
```