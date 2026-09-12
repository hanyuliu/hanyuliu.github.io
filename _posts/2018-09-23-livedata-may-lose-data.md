---
title: LiveData may lose data
date: 2018-09-23 00:00:00 -0800
categories: [Android]
tags: [android, livedata]
lang: en
---

> Originally published on [Medium](https://medium.com/@hanyuliu/livedata-may-lose-data-2fffdac57dc9) on Sep 23, 2018. Reposted here unchanged.
{: .prompt-info }

LiveData is one of the most notable star from Google's Android Architecture Component. Basically it is an observable data holder and dispatches data changes to observers when the observer is active. In Android App, views are usually attached to a lifecycle-aware component, like a Fragment or Activity. However, the lifecycle-aware component may be killed, recreated or detached(Fragment), which makes accessing views risky and complicated. To make view-layer operations more friendly, LiveData introduces lifecycle-awareness to its observers. Each observer can be associated to a LifecycleOwner, typically a Fragment or Activity, and prevents dispatching data change to the observer when its LifecycleOwner is inactive. This behavior could ensures the logic in observer will not be executed if the execution is not "necessary". For instance, if a fragment is detaching, its views will be destroyed soon. Any view-layer operations on this fragment's views during detaching will not be visible and necessary. Instead, LiveData buffers the last data change and notify the observer when the fragment is re-attached and views are recreated, which can "recover" the new view instances to the latest states and prevents potential NPE or memory leaks.

However, since LiveData dispatches data on main thread and only buffer one pending data change, there are few cases your observers may miss, or not receive, the data change. Although Google's documentation never defines LiveData as a RxJava-like streaming framework or recommend using LiveData other than a thin glue layer between View and VM, it is still necessary for us to study all potential data lost scenarios and avoid potential unexpected behaviors in production. In following section, lets discuss three data lost cases with examples.

## Emit data outside active state

LiveData is a lifecycle aware component. So the data change can only be delivered to an observer if its LifecycleOwner is active. As shown in following figure, a LifecycleOwner is active when it has switched to "STARTED" or "RESUMED". So a LiveData observer can only receive data change updates when its LifecycleOwner is "STARTED" or "RESUMED". If a data source emits data to a LiveData outside active states, observers will not receive data emitted.

![LiveData lifecycle states](/images/2018-09-23/lifecycle-diagram.jpeg)
_source: [android.jlelse.eu/exploring-livedata-architecture-component](https://android.jlelse.eu/exploring-livedata-architecture-component-f9375d3644ee)_

Suppose we have an Activity, which writes data to a MutableLiveData in its lifecycle callbacks. An observer is observing the MutableLiveData and prints out the data received.

```java
public class ActiveGapActivity extends BaseActivity {

    private static MutableLiveData<String> mutableLiveData = new MutableLiveData<>();

    @Override
    public void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mutableLiveData.observe(this, new Observer<String>() {
            @Override
            public void onChanged(@Nullable String string) {
                System.out.println("ActiveGapActivity Observed: " + string);
            }
        });
        mutableLiveData.setValue("onCreate");
    }

    @Override
    public void onResume() {
        super.onResume();
        mutableLiveData.setValue("onResume");

    }

    @Override
    public void onPause() {
        super.onPause();
        mutableLiveData.setValue("onPause");
    }

    @Override
    public void onStop() {
        super.onStop();
        System.out.println("ActiveGapActivity onStop called");
        mutableLiveData.setValue("onStop");
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        mutableLiveData.setValue("onDestroy");
    }
}
```

If we rotate the app and trigger an Activity recreation, following log will be printed:

```
ActiveGapActivity Observed: onCreate
ActiveGapActivity Observed: onResume
ActiveGapActivity Observed: onPause
ActiveGapActivity onStop called <== onStop() is called
ActiveGapActivity Observed: onCreate
ActiveGapActivity Observed: onResume
ActiveGapActivity Observed: onPause
ActiveGapActivity onStop called
ActiveGapActivity Observed: onCreate
ActiveGapActivity Observed: onResume
```

As shown in the log, the observer does not received "onStop", which is definitely called since "ActiveGapActivity onStop called" is printed. This behavior is caused by two LiveData proprieties:

1. LiveData can buffer one data instance;
2. LiveData will not notify observer during its LifecycleOwner is in "inactive" state.

When we rotate the screen, activity will be recreated. Current activity instance's onStop() and onDestroy() will be called sequentially. However, since the activity is no longer active, the data emitted will not be delivered to observer. Instead, the latest data instance is buffered internally, which is "onDestroy". When activity is recreated, "onCreate" is emitted by the second Activity instance and overrides the previous buffered data, "onDestroy". Once the activity, which is the LifecycleOwner, is back to active, the observer will only receive buffered "onCreate". Both "onStop" and "onDestroy" are lost.

Following figure shows the debug information of LiveData's setValue function when it is called in second activity's onCreate(). Clearly, new data instance, "onCreate", overrides the previous buffered data instance(the "mData"), which emitted by first instance's onDestroy().

![Debug view of LiveData.setValue overriding buffered mData](/images/2018-09-23/setvalue-debug.png)

## Emit data from background thread

LiveData dispatches data on main thread. If a source emits data from a background thread, down stream consumers will not receive update immediately. Instead, the data emitted will be buffered in LiveData and waiting for dispatching by main thread later. However, as LiveData only buffers one data instance, if the data source emits data "faster" than main thread can dispatch. The new data instance may overwrite the buffered one, which leads to data lost.

Lets check following example. An AsyncTask is kicked off in onResume() and emits three data instances to LiveData; an observer which prints out data is initialized in "onCreate".

```java
public class PostValueSampleActivity extends BaseActivity {

    private MutableLiveData<Integer> source;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        source = new MutableLiveData<>();
        source.observe(this, new Observer<Integer>() {
            @Override
            public void onChanged(@Nullable Integer integer) {
                append(integer.toString());
            }
        });
    }

    @Override
    protected void onResume() {
        super.onResume();


        AsyncTask.execute(new Runnable() {
            @Override
            public void run() {
                source.postValue(1);
                source.postValue(2);
                source.postValue(3);
            }
        });
    }
}
```

The example code will only print out "3", which is the last data emitted from AsyncTask. The magic behind this output can be found in LiveData's source code below.

When a background thread posted data to LiveData, LiveData will mark a flag, postTask, to true; and post a runnable to main thread to dispatch data change to observers. If the background thread emits data "too frequent"(before the main thread runnable finished), new data posted will override "mPendingData". When the runnable is executed by main thead and tries to dispatch the data change, it will get the last data, which is the "mPendingData" holds.

## Emit data in observer

Since LiveData dispatches data on main thread, when dispatching already started, everything is (kinds of) synchronized. All observers will be iterated and receive updates sequentially. However, if new data changes are set inside an observer, the new data changes(lets call it "B") will overwrite the one currently been dispatching(lets call it "A") and terminate current dispatching cycle. Some observers may not receive "A" forever.

Suppose we have two observers. Observer 2 prints what received, but observer 1 will set new values back to the LiveData if received number is even.

```java
public class SetValueSampleActivity extends BaseActivity {

    private MutableLiveData<Integer> source;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        source = new MutableLiveData<>();

        source.observe(this, new Observer<Integer>() {
            @Override
            public void onChanged(@Nullable Integer integer) {
                append("Receiver 1: " + integer.toString());
                if (integer % 2 == 0) {
                    source.setValue(integer * 10);
                    source.setValue(integer * 10 + 1);
                }
            }
        });
        source.observe(this, new Observer<Integer>() {
            @Override
            public void onChanged(@Nullable Integer integer) {
                append("Receiver 2: " + integer.toString());
            }
        });
    }

    @Override
    protected void onResume() {
        super.onResume();

        source.setValue(1);
        source.setValue(2);
        source.setValue(3);
    }
}
```

The output from this sample is shown in following figure. Observer 1 received 1, 2, 21, 3 and observer 2 received 1, 21, 3.

![Output of the two-observer sample](/images/2018-09-23/observer-output.png)

The reason behind this behavior is a little complicated. Lets check "dispatchingValue()" function in LiveData. As shown from line 2 to 6, if dispatching already started, any new data set will not kickoff another dispatching loop. Instead, new data change overrides pending data(in setValue() function), set "mDispatchInvalidated" to true and skip(from line 2 to 5). Later, "mDispatchInvalidated" will make dispatching while loop run one more time and dispatch the last data to observers. In our example, when observer 1 received "2", it set "20" and "21" back to LiveData. Since "mDispatchingValue" is true, "20" will not be dispatched and will be overrided by "21". After observer 1's logic is finished, as "mDispatchInvalidated" is true, it will break current dispatching loop(from line 13 to 19 and stops observer 2 receive "2") and make the outer while loop run one more time(from line 7 to 21). In this dispatching circle, all observers receive the last data, "21".

Full code samples can be found here: [hanyuliu/livedata_may_lose_data](https://github.com/hanyuliu/livedata_may_lose_data)
