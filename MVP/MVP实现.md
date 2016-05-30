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

##MvpLceView
对MvpView的基本封装。里面有五个抽象方法

<code> 

	public interface MvpLceView<M> extends MvpView {

	 
	  @UiThread
	  public void showLoading(boolean pullToRefresh);

	 
	  @UiThread
	  public void showContent();

	 
	  @UiThread
	  public void showError(Throwable e, boolean pullToRefresh);

	 
	  @UiThread
	  public void setData(M data);

	  @UiThread
	  public void loadData(boolean pullToRefresh);
	}

</code>

1、showLoading 在后台加载数据的时候显示loadding框。区分下拉刷新loadding和通用loadding
2、showContent 显示内容
3、showError 网络失败的。e为失败堆栈。 pullToRefresh 为是否需要收起下拉刷新。
4、setData M为需要显示的数据model
5、loadData 加载数据。 基本上在这个方法里面调用presenter去加载数据。


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

	  @UiThread
	  @Nullable public V getView() {
	    return viewRef == null ? null : viewRef.get();
	  }


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


##MvpNullObjectBasePresenter 
从 @ since 1.2.0  和类注释 我们能够知道 这个类是解决 null object的问题的。
检查传入的view是不是MvpView. 



<code>

	public abstract class MvpNullObjectBasePresenter<V extends MvpView> implements MvpPresenter<V> {

	  private WeakReference<V> view;
	  private final V nullView;

	  public MvpNullObjectBasePresenter() {
	    try {

	      // Scan the inheritance hierarchy until we reached MvpNullObjectBasePresenter
	      Class<V> viewClass = null;
	      Class<?> currentClass = getClass();

	      while (viewClass == null) {

	        Type genericSuperType = currentClass.getGenericSuperclass();

	        while (!(genericSuperType instanceof ParameterizedType)) {
	          // Scan inheritance tree until we find ParameterizedType which is probably a MvpSubclass
	          currentClass = currentClass.getSuperclass();
	          genericSuperType = currentClass.getGenericSuperclass();
	        }

	        Type[] types = ((ParameterizedType) genericSuperType).getActualTypeArguments();

	        for (int i = 0; i < types.length; i++) {
	          Class<?> genericType = (Class<?>) types[i];
	          if (genericType.isInterface() && isSubTypeOfMvpView(genericType)) {
	            viewClass = (Class<V>) genericType;
	            break;
	          }
	        }

	        // Continue with next class in inheritance hierachy (see genericSuperType assignment at start of while loop)
	        currentClass = currentClass.getSuperclass();
	      }

	      nullView = NoOp.of(viewClass);
	    } catch (Throwable t) {
	      throw new IllegalArgumentException(
	          "The generic type <V extends MvpView> must be the first generic type argument of class "
	              + getClass().getSimpleName()
	              + " (per convention). Otherwise we can't determine which type of View this"
	              + " Presenter coordinates.", t);
	    }
	  }

	  /**
	   * Scans the interface inheritnace hierarchy and checks if on the root is MvpView.class
	   *
	   * @param klass The leaf interface where to begin to scan
	   * @return true if subtype of MvpView, otherwise false
	   */
	  private boolean isSubTypeOfMvpView(Class<?> klass) {
	    if (klass.equals(MvpView.class)) {
	      return true;
	    }
	    Class[] superInterfaces = klass.getInterfaces();
	    for (int i = 0; i < superInterfaces.length; i++) {
	      if (isSubTypeOfMvpView(superInterfaces[0])) {
	        return true;
	      }
	    }
	    return false;
	  }

	  @UiThread @Override public void attachView(@NonNull V view) {
	    this.view = new WeakReference<V>(view);
	  }

	  @UiThread @NonNull protected V getView() {
	    if (view != null) {
	      V realView = view.get();
	      if (realView != null) {
	        return realView;
	      }
	    }

	    return nullView;
	  }

	  @UiThread @Override public void detachView(boolean retainInstance) {
	    if (view != null) {
	      view.clear();
	      view = null;
	    }
	  }
	}


</code>
构造函数里面有个方法，


NoOp.of(Class clazz) // Dynamically proxy to generate a new object instance for a given class by using reflections
用指定的class文件动态生成一个新的对象。