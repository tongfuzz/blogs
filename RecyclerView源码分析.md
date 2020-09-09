###RecyclerView源码分析

>  mCachedViews:用来一级缓存view，当超过两个时，取出第一个放入mScrapHeap中

>  mScrapHeap：二级缓存，当mCachedViews中存放的数量大于两个时，从mCachedViews取出第一个放入mScrapHeap中
