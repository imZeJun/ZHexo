---
title: View 绘制体系知识梳理(1) - LayoutInflater#inflate 源码解析
date: 2017-02-22 20:59
categories : View 绘制体系知识梳理
---
前几天在通过`LayoutInflater`渲染出子布局，并添加进入父容器的时候，出现了子布局的宽高属性不生效的情况，为此，总结一下和`LayoutInflater`相关的知识。
# 一、获得`LayoutInflater`
在`Android`当中，如果想要获得`LayoutInflater`实例，一共有以下3种方法：

## 1.1 `LayoutInflater inflater = getLayoutInflater();`
这种在`Activity`里面使用，它其实是调用了
```
    /**
     * Convenience for calling
     * {@link android.view.Window#getLayoutInflater}.
     */
    @NonNull
    public LayoutInflater getLayoutInflater() {
        return getWindow().getLayoutInflater();
    }
```
下面我们再来看一下`Window`的实现类`PhoneWindow.java`
```
    public PhoneWindow(Context context) {
        super(context);
        mLayoutInflater = LayoutInflater.from(context);
    }
```
它其实就是在构造函数中调用了下面`1.2`的方法。
而如果是调用了`Fragment`中也有和其同名的方法，但是是隐藏的，它的理由是：
```
    /**
     * @hide Hack so that DialogFragment can make its Dialog before creating
     * its views, and the view construction can use the dialog's context for
     * inflation.  Maybe this should become a public API. Note sure.
     */
    public LayoutInflater getLayoutInflater(Bundle savedInstanceState) {
        final LayoutInflater result = mHost.onGetLayoutInflater();
        if (mHost.onUseFragmentManagerInflaterFactory()) {
            getChildFragmentManager(); // Init if needed; use raw implementation below.
            result.setPrivateFactory(mChildFragmentManager.getLayoutInflaterFactory());
        }
        return result;
    }
```
## 1.2 `LayoutInflater inflater = LayoutInflater.from(this);`
```
    public static LayoutInflater from(Context context) {
        LayoutInflater LayoutInflater = (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);
        if (LayoutInflater == null) {
            throw new AssertionError("LayoutInflater not found.");
        }
        return LayoutInflater;
    }
```
可以看到，它其实是调用了`1.3`，但是加上了判空处理，也就是说我们从`1.1`当中的`Activity`和`1.2`方法中获取的`LayoutInflater`不可能为空。
## 1.3 `LayoutInflater LayoutInflater = (LayoutInflater) context.getSystemService(Context.LAYOUT_INFLATER_SERVICE);` 
这三种实现，默认最终都是调用了最后一种方式。
## 二、`LayoutInflater#inflate`
其`inflater`一共有四个重载方法，最终都是调用了最后一种实现。
## 2.1 `(@LayoutRes int resource, @Nullable ViewGroup root)`
```
    /**
     * Inflate a new view hierarchy from the specified xml resource. Throws
     * {@link InflateException} if there is an error.
     * 
     * @param resource ID for an XML layout resource to load (e.g.,
     *        <code>R.layout.main_page</code>)
     * @param root Optional view to be the parent of the generated hierarchy.
     * @return The root View of the inflated hierarchy. If root was supplied,
     *         this is the root View; otherwise it is the root of the inflated
     *         XML file.
     */
    public View inflate(@LayoutRes int resource, @Nullable ViewGroup root) {
        return inflate(resource, root, root != null);
    }
```
该方法，接收两个参数，一个是需要加载的`xml`文件的`id`，一个是该`xml`需要添加的布局，根据`root`的情况，返回值分为两种：
- 如果`root == null`，那么返回这个`root`
- 如果`root != null`，那么返回传入`xml`的根`View`

## 2.2 `(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot)`
```
    /**
     * Inflate a new view hierarchy from the specified xml node. Throws
     * {@link InflateException} if there is an error. *
     * <p>
     * <em><strong>Important</strong></em>   For performance
     * reasons, view inflation relies heavily on pre-processing of XML files
     * that is done at build time. Therefore, it is not currently possible to
     * use LayoutInflater with an XmlPullParser over a plain XML file at runtime.
     * 
     * @param parser XML dom node containing the description of the view
     *        hierarchy.
     * @param root Optional view to be the parent of the generated hierarchy.
     * @return The root View of the inflated hierarchy. If root was supplied,
     *         this is the root View; otherwise it is the root of the inflated
     *         XML file.
     */
    public View inflate(XmlPullParser parser, @Nullable ViewGroup root) {
        return inflate(parser, root, root != null);
    }
```
它的返回值情况和`2.1`类似，不过它提供的是不是`xml`的`id`，而是`XmlPullParser`，但是由于`View`的渲染依赖于`xml`在编译时的预处理，因此，这个方法并不合适。
## 2.3 `    public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot)`
```
    /**
     * Inflate a new view hierarchy from the specified xml resource. Throws
     * {@link InflateException} if there is an error.
     * 
     * @param resource ID for an XML layout resource to load (e.g.,
     *        <code>R.layout.main_page</code>)
     * @param root Optional view to be the parent of the generated hierarchy (if
     *        <em>attachToRoot</em> is true), or else simply an object that
     *        provides a set of LayoutParams values for root of the returned
     *        hierarchy (if <em>attachToRoot</em> is false.)
     * @param attachToRoot Whether the inflated hierarchy should be attached to
     *        the root parameter? If false, root is only used to create the
     *        correct subclass of LayoutParams for the root view in the XML.
     * @return The root View of the inflated hierarchy. If root was supplied and
     *         attachToRoot is true, this is root; otherwise it is the root of
     *         the inflated XML file.
     */
    public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
        final Resources res = getContext().getResources();
        if (DEBUG) {
            Log.d(TAG, "INFLATING from resource: \"" + res.getResourceName(resource) + "\" ("
                    + Integer.toHexString(resource) + ")");
        }

        final XmlResourceParser parser = res.getLayout(resource);
        try {
            return inflate(parser, root, attachToRoot);
        } finally {
            parser.close();
        }
    }
```
如果我们需要渲染的`xml`是`id`类型的，那么会先把它解析为`XmlResourceParser`，然后调用`2.4`的方法。
## 2.4 `public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot)`
```
    /**
     * Inflate a new view hierarchy from the specified XML node. Throws
     * {@link InflateException} if there is an error.
     * <p>
     * <em><strong>Important</strong></em>   For performance
     * reasons, view inflation relies heavily on pre-processing of XML files
     * that is done at build time. Therefore, it is not currently possible to
     * use LayoutInflater with an XmlPullParser over a plain XML file at runtime.
     * 
     * @param parser XML dom node containing the description of the view
     *        hierarchy.
     * @param root Optional view to be the parent of the generated hierarchy (if
     *        <em>attachToRoot</em> is true), or else simply an object that
     *        provides a set of LayoutParams values for root of the returned
     *        hierarchy (if <em>attachToRoot</em> is false.)
     * @param attachToRoot Whether the inflated hierarchy should be attached to
     *        the root parameter? If false, root is only used to create the
     *        correct subclass of LayoutParams for the root view in the XML.
     * @return The root View of the inflated hierarchy. If root was supplied and
     *         attachToRoot is true, this is root; otherwise it is the root of
     *         the inflated XML file.
     */
    public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
        synchronized (mConstructorArgs) {
            final Context inflaterContext = mContext;
            final AttributeSet attrs = Xml.asAttributeSet(parser);
            Context lastContext = (Context) mConstructorArgs[0];
            mConstructorArgs[0] = inflaterContext;
            View result = root;
            try {.
                int type;
                while ((type = parser.next()) != XmlPullParser.START_TAG &&
                        type != XmlPullParser.END_DOCUMENT) {
                    // Empty
                }
                //1.如果根节点的元素不是START_TAG，那么抛出异常。
                if (type != XmlPullParser.START_TAG) {
                    throw new InflateException(parser.getPositionDescription()
                            + ": No start tag found!");
                }
                final String name = parser.getName();
                //2.如果根节点的标签是<merge>，那么必须要提供一个root，并且该root要被作为xml的父容器。
                if (TAG_MERGE.equals(name)) {
                    if (root == null || !attachToRoot) {
                        throw new InflateException("<merge /> can be used only with a valid "
                                + "ViewGroup root and attachToRoot=true");
                    }
                    //2.1递归地调用它所有的孩子.
                    rInflate(parser, root, inflaterContext, attrs, false);
                } else {
                    //temp表示传入的xml的根View
                    final View temp = createViewFromTag(root, name, inflaterContext, attrs);
                    ViewGroup.LayoutParams params = null;
                    //如果提供了root并且不需要把xml的布局加入到其中，那么仅仅需要给它设置参数就好。
                    //如果提供了root并且需要加入，那么不会设置参数，而是调用addView方法。
                    if (root != null) {
                        //如果提供了root，那么产生参数。
                        params = root.generateLayoutParams(attrs);
                        if (!attachToRoot) {
                            temp.setLayoutParams(params);
                        }
                    }
                    //递归地遍历孩子.
                    rInflateChildren(parser, temp, attrs, true);
                    if (root != null && attachToRoot) {
                        //addView时需要加上前面产生的参数。
                        root.addView(temp, params);
                    }
                    //如果没有提供root，或者即使提供root但是不用将root作为parent，那么返回的是渲染的xml，在`root != null && attachToRoot`时，才会返回root。
                    if (root == null || !attachToRoot) {
                        result = temp;
                    }
                }

            } catch (XmlPullParserException e) {
                InflateException ex = new InflateException(e.getMessage());
                ex.initCause(e);
                throw ex;
            } catch (Exception e) {
                InflateException ex = new InflateException(
                        parser.getPositionDescription()
                                + ": " + e.getMessage());
                ex.initCause(e);
                throw ex;
            } finally {
                // Don't retain static reference on context.
                mConstructorArgs[0] = lastContext;
                mConstructorArgs[1] = null;
            }
            return result;
        }
    }
```
我们简单总结一下英文注释当中的说明，具体的流程可以看上面的中文注释。
- 作用：从特定的`xml`节点渲染出一个新的`view`层级。
- 提示：为了性能考虑，不应当在运行时使用`XmlPullParser`来渲染布局。
- 参数`parser`：包含有描述`xml`布局层级的`parser xml dom`。
- 参数`root`，可以是渲染的`xml`的`parent`（`attachToRoot == true`），或者仅仅是为了给渲染的`xml`层级提供`LayoutParams`。
- 参数`attachToRoot`：渲染的`View`层级是否被添加到`root`中，如果不是，那么**仅仅为`xml`的根布局生成正确的`LayoutParams`**。
- 返回值：**如果`attachToRoot`为真，那么返回`root`**，否则返回渲染的`xml`的根布局。

## 三、不指定root的情况

由前面的分析可知，当我们没有传入`root`的时候，`LayoutInflater`不会调用`temp.setLayoutParams(params)`，也就是像之前我遇到问题时的使用方式一样：
```
LinearLayout linearLayout = (LinearLayout) mLayoutInflater.inflate(R.layout.linear_layout, null);
mContentGroup.addView(linearLayout);
```
当没有调用上面的方法时，`linearLayout`内部的`mLayoutParams`参数是没有被赋值的，下面我们再来看一下，通过这个返回的`temp`参数，把它通过不带参数的`addView`方法添加进去，会发生什么。
调用`addView`后，如果没有指定`index`，那么会把`index`设为`-1`，按前面的分析，那么下面这段逻辑中的`getLayoutParams()`必然是返回空的。
```
    public void addView(View child, int index) {
        if (child == null) {
            throw new IllegalArgumentException("Cannot add a null child view to a ViewGroup");
        }
        LayoutParams params = child.getLayoutParams();
        if (params == null) {
            params = generateDefaultLayoutParams();
            if (params == null) {
                throw new IllegalArgumentException("generateDefaultLayoutParams() cannot return null");
            }
        }
        addView(child, index, params);
    }
```
在此情况下，为它提供了默认的参数，那么，这个默认的参数是什么呢？
```
protected LayoutParams generateDefaultLayoutParams() {
    return new LayoutParams(LayoutParams.WRAP_CONTENT, LayoutParams.WRAP_CONTENT);
}
```
也就是说，当我们通过上面的方法得到一个`View`树之后，将它添加到某个布局中，这个`View`数所指定的根布局中的宽高属性其实是不生效的，而是变为了`LayoutParams.WRAP_CONTENT, LayoutParams.WRAP_CONTENT`。
