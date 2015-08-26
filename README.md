## 一个android英文有声读物app

在得到了比较完整的timing file之后，剩下的工作是，如何呈现它。
**初步构想是按页呈现文本，支持自动翻页和手动翻页，通过ViewPager和Fragment实现**。所以问题是：
给定一个文本，如何拆分成页（每页铺满屏幕）？
比如可以拆分为3页，第1页包含21个单词，第2页包含22个单词，第3页包含23个单词。如果得到这些数据，那么就非常容易构建界面了。

### 构建拆分文本的Adapter

```java
public class ChapterPageAdapter extends FragmentPagerAdapter {
    // ViewPager的adapter的构造器，以文本文件的Uri作为参数。
    public ChapterPageAdapter(Context context, FragmentManager fm, Uri jsonUri) {
        super(fm);
        mAppContext = context;
        assert context != null;
        mJsonUri = jsonUri;
        mPageBeginningWordIndexList = new ArrayList<Integer>();
        mPageBeginningWordTimmingList = new ArrayList<Integer>();
        splitChapterToPages(mJsonUri);
    }
    // 在拿到文本后，通过下面两个private函数拆分文本，
    // 而拆分的关键是，根据屏幕的宽度、高度，每个单词的宽度，计算一个屏幕可以显示的单词个数，用到的一些计算宽高度的方法被抽象成一个独立的类`ChapterPageUtil.class`，
    // 最终得到每页首单词在文本中的序号。
    private void splitChapterToPages(Uri jsonUri) {}
    private int totalWordsCanDisplayOnOnePage(List<String> words, int startIndex) {}

    @Override
    public int getCount() {
        // ViewPager托管的Fragment个数。
        return 首单词序号的个数;
    }

    @Override
    public Fragment getItem(int poisition) {
        ......
        // 这就是ViewPager第position页需要显示的内容。
        ChapterPageFragment fragment = ChapterPageFragment.newInstance(uri, fromIndex, count);
        return fragment;
    }
```

### 每页是一个Fragment

```java
public class ChapterPageFragment extends Fragment implements OnClickListener {
    // Create fragment instance
    // 每页对应一个Fragment，需要告知它要显示的文本，从哪个单词开始显示，总共显示几个单词。
    public static ChapterPageFragment newInstance(Uri jsonUri, int fromIndex, int count) {
        Bundle args = new Bundle();
        args.putString(EXTRA_TIMING_JSON_URI, jsonUri.toString());
        args.putInt(EXTRA_FROM_INDEX, fromIndex);
        args.putInt(EXTRA_COUNT, count);
        
        ChapterPageFragment fragment = new ChapterPageFragment();
        fragment.setArguments(args);
        return fragment;
    }
    
    // Fragment是LinearLayout布局，
    // 每个单词对应一个TextView，添加到 sub LinearLayout，填满后再添加到下一个 sub LinearLayout（对应下一行）；
    // 每个换行符独占一个 sub LinearLayout。
    public View onCreateView(LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState) {}
```

### 实例化文本为一个singleton

考虑到ViewPager Adapter和所有的Fragment都要用到文本信息，我们把文本实例化为singleton，便于文本数据取用。

```java
public class TalkingBookChapter {
    // Singletons and centralized data storage
    private static TalkingBookChapter sChapter;

    // Setting up the singleton
    public static TalkingBookChapter get(Context context, Uri timingJsonUri) {
        if (sChapter == null) {
            sChapter = new TalkingBookChapter(context, timingJsonUri);
        }
        return sChapter;
    }

    private TalkingBookChapter(Context context, Uri timingJsonUri) {}
    // 下面两个函数就是为了Fragment取它所需要显示的文本。
    public List<String> getWordList(int fromIndex, int count) {}    
    public List<Integer> getTimingList(int fromIndex, int count) {}    
    public int size() {}
```

### 如何在点击单词或者翻页时跳转到对应的音频位置

Fragment定义一个interface，在TextView被点击时调用，Activity实现它完成音频位置的调节。
翻页的时候，就更简单了，只需要override ViewPager的OnPageChangeListener.

```java
public class ChapterPageFragment extends Fragment implements OnClickListener {
    private OnWordClickListener mOnWordClickListener;    

    public void setOnWordClickListener(OnWordClickListener l) {
        mOnWordClickListener = l;
    }

    public interface OnWordClickListener {
        public void onWordClick(int msec);
    }

    @Override
    public void onClick(View v) {
        if (v instanceof TextView) {
            int msec = (int)v.getTag();
            if (mOnWordClickListener != null) {
                mOnWordClickListener.onWordClick(msec);
            }
        }
    }
}

public class FullScreenPlayerActivity extends FragmentActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        mChapterPageAdapter = new ChapterPageAdapter(this, getSupportFragmentManager(), mTimingJsonUri);
        mChapterPageAdapter.setOnChapterPageWordClickListener(mOnPageAdapterWordClickListener);
        mChapterViewPager.setAdapter(mChapterPageAdapter);

    // 这个Activity管理ViewPager，Fragment定义的interface被ViewPager的Adapter又包了一层，
    // 所以当Fragment的TextView被点击后，最终会调用到这里：
    private OnChapterPageWordClickListener mOnPageAdapterWordClickListener = new OnChapterPageWordClickListener() {
        @Override
        public void onChapterPageWordClick(int msec) {
            seekToPosition(msec); // 这个函数调用到 MediaPlayer.seekTo(int msec)，调节到指定的音频位置。
        }
    };

    private ViewPager.OnPageChangeListener mOnPageChangeListener = new OnPageChangeListener() {
        @Override
        public void onPageSelected(int position) {
                ......
                seekToPosition(mChapterPageAdapter.getPageTiming(position)); // 翻页时，调节到指定的音频位置。
        }
```

### 如何高亮当前读到的文本

```java
public class FullScreenPlayerActivity extends FragmentActivity {
    // 这是一个更新界面的定时任务，
    private void updateProgress() {
        runOnUiThread(new Runnable() {
            @Override
            public void run() {
                int msec = mPlayerController.getCurrentPosition();
                mChapterPageAdapter.seekChapterToTime(msec);
                // check if seeking time is out of selected page, if true, then set ViewPager to current item.
                // 如果已经读到下一页，那么执行ViewPager.setCurrentItem()
                ......
            }
        });
    }

public class ChapterPageAdapter extends FragmentPagerAdapter {
    // This method will be called to notify fragment to update view in order to highlight the reading word.
    public void seekChapterToTime(int msec) {
        mSeekingTime = msec;
        notifyDataSetChanged(); // 调用这个方法后，getItemPosition()会被调用，我们在getItemPosition()中完成Fragment界面的更新。
    }

    @Override
    public int getItemPosition(Object object) {
        if (object instanceof ChapterPageFragment) {
            // This method called to update view in order to highlight the reading word.
            ((ChapterPageFragment) object).seekChapterToTime(mSeekingTime);
        }
        return super.getItemPosition(object);
    }
```

### 关于章节列表对应的类的说明 TODO
### 关于播放器的说明 TODO


## 用到的开源软件

- android-UniversalMusicPlayer
这是一个开源的android音乐播放器，[它的项目主页](https://github.com/googlesamples/android-UniversalMusicPlayer)
我的播放器代码很多直接拿了它的一个文件 [FullScreenPlayerActivity.java 点击查看](https://github.com/googlesamples/android-UniversalMusicPlayer/blob/master/mobile/src/main/java/com/example/android/uamp/ui/FullScreenPlayerActivity.java)

- ViewPagerIndicator
这是一个开源的Android UI，用以标示ViewPager的页，就是常见的几个小圆点。
[它的项目主页](https://github.com/JakeWharton/ViewPagerIndicator)

- audiosync
这个就是音频文本同步的开源命令行工具。[它的项目主页](https://github.com/johndyer/audiosync)。
但是呢，它包含上百兆的音频文件，在时好时坏的国外网站访问现实下，有时只有几kb的下载速度，我就folk它然后删掉它的音频文件。这样子。


## 去哪下载

2015年08月21日完成了1.0版本，App名字叫**TalkingBook21**（因为我叫li21嘛），[你可以在这里下载App](http://pan.baidu.com/s/1kT3rI1h)，
由于音频文件很大，所以只在app里包了一个音频，权当是个demo。
[更多的音频需要在这里下载](http://pan.baidu.com/s/1bnyivnT)，目前仅实现了《麦田的守望者 The Catcher in the Rye》的同步，音频总时长7个小时，293M。
所以坦白的讲，这个app实质上是**麦田的守望者音文同步有声读物Android App**.

你需要把它解压后放入手机的外置SD卡：YourExtSDCard/TalkingBook21/，然后重新启动App（彻底杀掉！），重启后的App会向你展示章节列表，点击即可播放。

[这里是app的源码](https://github.com/li2/TalkingBook21_AndroidApp)
[这里是制作timing json file的开源命令行工具](https://github.com/li2/TalkingBook21_AudioSync)

![DemoImage](https://github.com/li2/TalkingBook21_AndroidApp/blob/master/DemoImage.png)
