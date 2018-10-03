---
layout: post
title: Load and play video file from device android
image: /img/gallery.png
---

We can do this by using `LoaderManager` with callback, however it seems deprecated both `support` or `non-support` version from `android-O`

So, Let's do it by using `ContentResolver`

# FOR LOADING ALL VIDEOS FROM DEVICES
## Create a contract
This help you separate the content that you need, and easily to 
```Java
class LocalVideoContract {
    companion object {
        const val VIDEO_NAME = MediaStore.Video.Media.DISPLAY_NAME
        const val VIDEO_PATH = MediaStore.Video.Media.DATA
        const val VIDEO_TYPE = MediaStore.Video.Media.MIME_TYPE
        const val VIDEO_DURATION = MediaStore.Video.Media.DURATION
        const val VIDEO_SIZE = MediaStore.Video.Media.SIZE
        const val VIDEO_DATE_ADDED = MediaStore.Video.Media.DATE_ADDED
        const val VIDEO_ALBUM = MediaStore.Video.Media.ALBUM
        const val VIDEO_RESOLUTION = MediaStore.Video.Media.RESOLUTION

        // The Uri path
        fun queryUri(): Uri = MediaStore.Video.Media.EXTERNAL_CONTENT_URI

        fun projection() = arrayOf(VIDEO_NAME, VIDEO_PATH, VIDEO_TYPE, VIDEO_DURATION, VIDEO_SIZE, VIDEO_DATE_ADDED, VIDEO_ALBUM, VIDEO_RESOLUTION)

        fun selection(): String {
            val argsSize = selectionArgs().size
            val str = StringBuilder()
            for (i in 0 until argsSize) {
                if (i > 0) {
                    str.append(",")
                }
                str.append("?")
            }
            // For support load multi types of video
            return VIDEO_TYPE.plus(" in (").plus(str.toString()).plus(")")
        }

        fun selectionArgs() = arrayOf("video/mp4", "video/m4v", "video/mkv", "video/mov", "video/avi", "video/ts", "video/webm")

        // Order by Last modified
        fun queryOrder() = "$VIDEO_DATE_ADDED DESC"
    }
}
```
## Load the videos
```Java
// Note that LocalVideoData is your defined model of local video
fun fetchVideos(): Single<List<LocalVideoData>> {
    return Single.create<List<LocalVideoData>> {
        try {
            val cursor = contentResolver.query(queryUri(), projection(), selection(), selectionArgs(), queryOrder())
            val videosList = mutableListOf<LocalVideoData>()
            var idx = 0
            if (cursor.moveToFirst()) {
                do {
                    videosList.add(LocalVideoData(
                        cursor.getString(cursor.getColumnIndex(VIDEO_NAME)),
                        cursor.getString(cursor.getColumnIndex(VIDEO_PATH)),
                        cursor.getString(cursor.getColumnIndex(VIDEO_TYPE)),
                        cursor.getLong(cursor.getColumnIndex(VIDEO_DURATION)),
                        cursor.getLong(cursor.getColumnIndex(VIDEO_SIZE)),
                        cursor.getLong(cursor.getColumnIndex(VIDEO_DATE_ADDED)),
                        cursor.getString(cursor.getColumnIndex(VIDEO_ALBUM)),
                        cursor.getString(cursor.getColumnIndex(VIDEO_RESOLUTION))))
                    idx++
                } while (cursor.moveToNext())
            }
            cursor.close()
            it.onSuccess(videosList)
        } catch (ex: Exception) {
            it.onError(ex)
        }
    }
}
```
# PLAY THE VIDEO USING VIDEOVIEW
From your layout
```XML
<VideoView
    android:id="@+id/video_view"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_gravity="center"
    />
<!-- Should noted that width and height should 1 of them has fixed size, then we can easily set the ratio of video fram base on its size-->
```
From your Kotlin/Java code
```Java
private var uiHandler = Handler(Looper.getMainLooper())
private fun prepareVideo(videoPath: String) {
    video_view.setVideoPath(videoPath)
    video_view.setOnPreparedListener {
        // Enable your control after video is prepared
        btnPlay.isEnabled = true 
        btnPlay.safeClick(View.OnClickListener {
            handlePlayVideo()
        })
        //... 
        
        // Scaling the video image for keep its ratio
        it.setVideoScalingMode(MediaPlayer.VIDEO_SCALING_MODE_SCALE_TO_FIT)

        // Set the first image to video view
        val mediaRetriever = MediaMetadataRetriever()
        mediaRetriever.setDataSource(videoPath)
        try {
            val bitmap = mediaRetriever.getFrameAtTime(video_view.currentPosition.toLong())
            video_view.background = BitmapDrawable(resources, bitmap)
        } catch (e: Exception) {
            // Hack or Trick if bitmap is not created
            video_view.seekTo(100)
        }

        // Handle for video length time and played time
        totalTime.text = video_view.duration // do format you want 00:00
        playedTime.text = video_view.currentPosition 
        seekbarPlay.max = video_view.duration
    }

    video_view.setOnCompletionListener {
        // reset Player and related UI when video is completed
        resetPlayer()
    }
}
private fun handlePlayVideo() {
    if (video_view.isPlaying) {
        handlePaused()
    } else {
        handlePlayed()
    }
}

private fun handlePaused() {
    video_view.pause()
    uiHandler.removeCallbacksAndMessages(null)
}

private fun handlePlayed() {
    if (video_view.isPlaying) {
        video_view.resume()
    } else {
        video_view.seekTo(video_view.currentPosition)
        video_view.start()
        video_view.background = null // Remember to remove the first bitmap
    }

    uiHandler.postDelayed(seekBarRunnable, SEEK_BAR_UPDATE_INTERVAL)
    uiHandler.postDelayed(playedTimeRunnable, PLAYED_TIME_UPDATE_INTERVAL)
}


private val seekBarRunnable = object : Runnable {
    override fun run() {
        if (!video_view.isPlaying) return
        seekbarPlay.progress = video_view.currentPosition
        uiHandler.postDelayed(this, SEEK_BAR_UPDATE_INTERVAL)

    }
}

private val playedTimeRunnable = object : Runnable {
    override fun run() {
        if (!video_view.isPlaying) return
        playedTime.text = video_view.currentPosition 
        uiHandler.postDelayed(this, PLAYED_TIME_UPDATE_INTERVAL)
    }
}
```
