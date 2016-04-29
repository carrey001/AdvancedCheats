#mvp

 <br>

一个偶然的机会获知了一个mvp框架库。[一个三方库](https://github.com/sockeqwe/mosby) 。 我大致看了下 封装的很不错。我们就看看这里面的封装代码来学习学习mvp框架。

  看我们看到了 map-common 库里面就几几个文件，我们逐一查看这些文件。

## MvpView
<code>
	/**
 * The root view interface for every mvp view
 *
 * @author Hannes Dorfmann
 * @since 1.0.0
 */
public interface MvpView {
}
</code>

看doc 描述可有知道 mvpView 是作为父view 来管理所有的mvpView

##MvpPresenter

<code>
/**
 * The base interface for each mvp presenter.  ///，父Presenter 
 *
 * Mosby assumes that all interaction (i.e. updating the View) between Presenter and View is
 * executed on android's main UI thread.
 *
 * @author Hannes Dorfmann
 * @since 1.0.0
 */
public interface MvpPresenter<V extends MvpView> {

  /**
   * Set or attach the view to this presenter  //把MvpView 添加到Presenter 
   */
  @UiThread
  void attachView(V view);

  /**
   * Will be called if the view has been destroyed. Typically this method will be invoked from
   * Activity.detachView()or Fragment.onDestroyView() //释放MvpView  
   */
  @UiThread
  void detachView(boolean retainInstance);
}
</code>

把Presenter  添加（释放）MvpView

## MvpBasePresenter

<code> 


public class MvpBasePresenter<V extends MvpView> implements MvpPresenter<V> {

  private WeakReference<V> viewRef;

  @UiThread
  @Override public void attachView(V view) {
    viewRef = new WeakReference<V>(view);
  }

  /**
   * Get the attached view. You should always call {@link #isViewAttached()} to check if the view
   * is
   * attached to avoid NullPointerExceptions.
   *
   * @return <code>null</code>, if view is not attached, otherwise the concrete view instance
   */
  @UiThread
  @Nullable public V getView() {
    return viewRef == null ? null : viewRef.get();
  }

  /**
   * Checks if a view is attached to this presenter. You should always call this method before
   * calling {@link #getView()} to get the view instance.
   */
  @UiThread
  public boolean isViewAttached() {
    return viewRef != null && viewRef.get() != null;
  }

  @UiThread
  @Override public void detachView(boolean retainInstance) {
    if (viewRef != null) {
      viewRef.clear();
      viewRef = null;
    }
  }
}
</code>

基本mvpPresenter 封装了获取MvpView 和释放MvpView   对于MvpView 使用弱引用。避免内存泄漏。