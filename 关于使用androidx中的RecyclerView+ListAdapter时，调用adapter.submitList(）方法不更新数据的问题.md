###关于使用androidx中的RecyclerView+ListAdapter时，调用adapter.submitList(）方法不更新数据的问题

在androidx中，RecyclerView为我们提供了ListAdapter类用来展示数据，它RecyclerView.Adapter功能进行了扩展，提供了计算adapter中的老数据和新数据不同的功能，此计算过程是在后台线程中进行的。

在创建ListAdatper类时，需要传递一个DiffUtil.ItemCallback<T>类对象，DiffUtil.ItemCallback<T>是一个抽象类，其中包含了三个方法

```java
public abstract static class ItemCallback<T> {
        //用来判断两个条目是否时相同条目，通常相同id的条目认定为相同
        public abstract boolean areItemsTheSame(@NonNull T oldItem, @NonNull T newItem);
		//判断两个条目内容是否相同
        public abstract boolean areContentsTheSame(@NonNull T oldItem, @NonNull T newItem);
		//当areItemsTheSame返回true，areContentsTheSame返回false时调用此方法，用来更新数据，我们可以将不同的数据添加到bundle对象中，并在三个参数的onBindViewHolder方法中获取不同的地方，并更新数据
        @SuppressWarnings({"WeakerAccess", "unused"})
        @Nullable
        pub lic Object getChangePayload(@NonNull T oldItem, @NonNull T newItem) {
            return null;
        }
    }
```

在我们设置完adapter后，为adapter设置数据时调用adapter.submitList(list）方法，但是如果设置的list对象与上一个对象是同一个对象时并不会更新数据，这点可以通过源码可以看出，google这么做可能是考虑到使用Room等数据库管理软件，每次获取数据都是新的list对象

```java
public void submitList(@Nullable final List<T> newList) {
        ...
        if (newList == mList) {
            //可以看到如果list对象是同一个对象的话，是不会做任何操作的
            return;
        }
        ...
    }
```

但是我们平常都是使用一个List对象来接收数据，当加载更多数据时也是通过addAll()方法向其中添加，那么当我们添加更多数据时再调用adapter.submitList(list）方法并不会更新列表，此时只有从新创建一个List对象并将其设置给adapter才会起作用，每次创建list对象并拷贝数据比较麻烦，我们可以使用kotlin提供的扩展方法

```java
public fun <T> Collection<T>.toMutableList(): MutableList<T> {
    return ArrayList(this)
}
```
可以看到，只要是Collection对象，只要调用toMutableList()就会重新创建一个List对象，所以我们在更新数据时只需要如下调用即可

```java
adapter.submitList(list.toMutableList())
```
