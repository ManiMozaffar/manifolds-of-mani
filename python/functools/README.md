## lru_cache

```python
@functools.lru_cache(maxsize=None)
def fibonacci(n):
    if n <= 1:
        return n
    else:
        return fibonacci(n-1) + fibonacci(n-2)
```


اولین موردی که بهش میپردازم از functools هست که میخوام یک سیستم کش رو براش بسازم
ولی با این تفاوت که بر اساس ورودی های تابع باشه.

اینو خیلی دوست دارم چون براتون ریزالتو کش میکنه ولی با این تفاوت که بر اساس args که بهش میدین این کش انجام میشه.

حالا یعنی چی, یعنی وقتی شما fibonacci(50) رو صدا میزنید 50امین عدد عدد fibonacci حساب میشه, و بار بعدی که fibonacci(50)  رو صدا میزنید دیگه نمیره محاسبه کنه و تو کش ذخیره میکنه که fibonacci(50)  مساوی با اون عدده.

نکته جالب تر و تمیزتر اینه که حتی fibonacci(49)  و ... هم ذخیره میکنه. یعنی شما دیگه fibonacci(51)  هم صدا بزنید خیلی سریعتر ران میشه.  

یک arg هم داره که به اسم maxsize, که درواقع مشخص میکنه ماکسیموم کش اختصاص داده شده برای این تابع چقدر باشه. 
به طور دیفالت هم 128 هست. حواستون باشه مثل مثال بالا اگه از None استفاده کنید ممکنه مشکل پر شدن حافظه براتون رخ بده. 



```python
def _lru_cache_wrapper(user_function, maxsize, typed, _CacheInfo):
# Constants shared by all lru cache instances:
sentinel = object()          # unique object used to signal cache misses
make_key = _make_key         # build a key from the function arguments
PREV, NEXT, KEY, RESULT = 0, 1, 2, 3   # names for the link fields

cache = {}
hits = misses = 0
full = False
cache_get = cache.get    # bound method to lookup a key or return None
cache_len = cache.__len__  # get cache size without calling len()
lock = RLock()           # because linkedlist updates aren't threadsafe
root = []                # root of the circular doubly linked list
root[:] = [root, root, None, None]     # initialize by pointing to self

if maxsize == 0:

    def wrapper(*args, **kwds):
        # No caching -- just a statistics update
        nonlocal misses
        misses += 1
        result = user_function(*args, **kwds)
        return result

elif maxsize is None:

    def wrapper(*args, **kwds):
        # Simple caching without ordering or size limit
        nonlocal hits, misses
        key = make_key(args, kwds, typed)
        result = cache_get(key, sentinel)
        if result is not sentinel:
            hits += 1
            return result
        misses += 1
        result = user_function(*args, **kwds)
        cache[key] = result
        return result

else:

    def wrapper(*args, **kwds):
        # Size limited caching that tracks accesses by recency
        nonlocal root, hits, misses, full
        key = make_key(args, kwds, typed)
        with lock:
            link = cache_get(key)
            if link is not None:
                # Move the link to the front of the circular queue
                link_prev, link_next, _key, result = link
                link_prev[NEXT] = link_next
                link_next[PREV] = link_prev
                last = root[PREV]
                last[NEXT] = root[PREV] = link
                link[PREV] = last
                link[NEXT] = root
                hits += 1
                return result
            misses += 1
        result = user_function(*args, **kwds)
        with lock:
            if key in cache:
                # Getting here means that this same key was added to the
                # cache while the lock was released.  Since the link
                # update is already done, we need only return the
                # computed result and update the count of misses.
                pass
            elif full:
                # Use the old root to store the new key and result.
                oldroot = root
                oldroot[KEY] = key
                oldroot[RESULT] = result
                # Empty the oldest link and make it the new root.
                # Keep a reference to the old key and old result to
                # prevent their ref counts from going to zero during the
                # update. That will prevent potentially arbitrary object
                # clean-up code (i.e. __del__) from running while we're
                # still adjusting the links.
                root = oldroot[NEXT]
                oldkey = root[KEY]
                oldresult = root[RESULT]
                root[KEY] = root[RESULT] = None
                # Now update the cache dictionary.
                del cache[oldkey]
                # Save the potentially reentrant cache[key] assignment
                # for last, after the root and links have been put in
                # a consistent state.
                cache[key] = oldroot
            else:
                # Put result in a new link at the front of the queue.
                last = root[PREV]
                link = [last, root, key, result]
                last[NEXT] = root[PREV] = cache[key] = link
                # Use the cache_len bound method instead of the len() function
                # which could potentially be wrapped in an lru_cache itself.
                full = (cache_len() >= maxsize)
        return result
```

Let's dig in a bit :) 🐕

سه حالت برای کش وجود داره,
در حالت اول، هیچ کشی انجام نمیشه و تنها آمار کلی از تعداد بارهایی که تابع فراخوانی شده ثبت میشه.
در حالت دوم، کش ساده‌ای بدون محدودیت سایز وجود دارد. در این حالت، نتیجه‌ی تابع برای ورودی‌های مشابه ذخیره شده و برای بارهای بعدی فراخوانی می‌شود.
در حالت سوم، سایز کش محدود شده است. در این حالت، نتایج قدیمی‌تر ترک می‌شوند و به جای آن‌ها، نتایج جدیدی که اخیرا استفاده شده‌اند، ذخیره می‌شوند. پس وقتی maxsize پر شد بهتون ارور نمیده :)). جالبه نه؟