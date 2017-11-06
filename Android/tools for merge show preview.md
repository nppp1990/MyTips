tools for merge show preview

https://stackoverflow.com/questions/17296552/preview-layout-with-merge-root-tag-in-intellij-idea-android-studio

~~~xml
<merge xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:parentTag="LinearLayout"
    tools:orientation="horizontal">

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Some text"
        android:textSize="20sp"/>

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Some other text"/>
</merge>
~~~

