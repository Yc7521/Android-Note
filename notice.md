# 笔记

## 通知

### 创建基本通知

最基本、精简形式（也称为折叠形式）的通知会显示一个图标、一个标题和少量内容文本。在本部分中，将了解如何创建用户点击后可启动应用中的 Activity 的通知。

### 设置通知内容

- 小图标，通过 `setSmallIcon()` 设置。这是所必需的唯一用户可见内容。
- 标题，通过 `setContentTitle()` 设置。
- 正文文本，通过 `setContentText()` 设置。
- 通知优先级，通过 `setPriority()` 设置。优先级确定通知在 Android 7.1 和更低版本上的干扰程度。

```java
NotificationCompat.Builder builder
    = new NotificationCompat
        .Builder(this, CHANNEL_ID)
        .setSmallIcon(R.drawable.notification_icon)
        .setContentTitle(textTitle)
        .setContentText(textContent)
        .setPriority(NotificationCompat.PRIORITY_DEFAULT);
```

### 设置通知的点按操作

每个通知都应该对点按操作做出响应，通常是在应用中打开对应于该通知的 Activity。为此，必须指定通过 `PendingIntent` 对象定义的内容 Intent，并将其传递给 `setContentIntent()`。

```java
// Create an explicit intent for an Activity in your app
Intent intent = new Intent(this, AlertDetails.class);
intent.setFlags(
    Intent.FLAG_ACTIVITY_NEW_TASK
    | Intent.FLAG_ACTIVITY_CLEAR_TASK);
PendingIntent pendingIntent
    = PendingIntent.getActivity(this, 0, intent, 0);

NotificationCompat.Builder builder
    = new NotificationCompat
        .Builder(this, CHANNEL_ID)
        .setSmallIcon(R.drawable.notification_icon)
        .setContentTitle("My notification")
        .setContentText("Hello World!")
        .setPriority(NotificationCompat.PRIORITY_DEFAULT)
// Set the intent that will fire when the user taps the notification
        .setContentIntent(pendingIntent)
        .setAutoCancel(true);
```

调用 `setAutoCancel()`，它会在用户点按通知后自动移除通知。

### 显示通知

调用 `NotificationManagerCompat.notify()`，并将通知的唯一 ID 和 `NotificationCompat.Builder.build()` 的结果传递给它。

```java
NotificationManagerCompat notificationManager
    = NotificationManagerCompat.from(this);
// notificationId is a unique int for each notification that you must define
notificationManager.notify(notificationId, builder.build());
```

需要保存传递到 `NotificationManagerCompat.notify()` 的通知 ID，因为如果之后想要更新或移除通知，将需要使用这个 ID。

### 添加操作按钮

一个通知最多可以提供三个操作按钮，让用户能够快速响应，例如暂停提醒，甚至回复短信。但这些操作按钮不应该重复用户在点按通知时执行的操作。

如需添加操作按钮，请将 `addAction()` 传递给 `PendingIntent` 方法。这就像在设置通知的默认点按操作，不同的是不会启动 Activity，而是可以完成各种其他任务，例如启动在后台执行作业的 `BroadcastReceiver`，这样该操作就不会干扰已经打开的应用。

例如，以下代码演示了如何向特定接收者发送广播：

```java
Intent snoozeIntent = new Intent(this, MyBroadcastReceiver.class);
snoozeIntent.setAction(ACTION_SNOOZE);
snoozeIntent.putExtra(EXTRA_NOTIFICATION_ID, 0);
PendingIntent snoozePendingIntent =
    PendingIntent.getBroadcast(this, 0, snoozeIntent, 0);

NotificationCompat.Builder builder
     = new NotificationCompat
        .Builder(this, CHANNEL_ID)
        .setSmallIcon(R.drawable.notification_icon)
        .setContentTitle("My notification")
        .setContentText("Hello World!")
        .setPriority(NotificationCompat.PRIORITY_DEFAULT)
        .setContentIntent(pendingIntent)
        .addAction(R.drawable.ic_snooze, getString(R.string.snooze),
                snoozePendingIntent);
```

### 添加进度条

通知可以包含动画形式的进度指示器，向用户显示正在进行的操作的状态。

如果可以估算操作在任何时间点的完成进度，应通过调用 `setProgress(max, progress, false)` 使用指示器的“确定性”形式。第一个参数是“完成”值（如 100）；第二个参数是当前完成的进度，最后一个参数表明这是一个确定性进度条。
随着操作的继续，持续使用 `progress` 的更新值调用 `setProgress(max, progress, false)` 并重新发出通知。

```java
...
NotificationManagerCompat notificationManager
    = NotificationManagerCompat.from(this);
NotificationCompat.Builder builder
    = new NotificationCompat.Builder(this, CHANNEL_ID);
builder.setContentTitle("Picture Download")
        .setContentText("Download in progress")
        .setSmallIcon(R.drawable.ic_notification)
        .setPriority(NotificationCompat.PRIORITY_LOW);

// Issue the initial notification with zero progress
int PROGRESS_MAX = 100;
int PROGRESS_CURRENT = 0;
builder.setProgress(PROGRESS_MAX, PROGRESS_CURRENT, false);
notificationManager.notify(notificationId, builder.build());

// Do the job here that tracks the progress.
// Usually, this should be in a
// worker thread
// To show progress, update PROGRESS_CURRENT and update the notification with:
// builder.setProgress(PROGRESS_MAX, PROGRESS_CURRENT, false);
// notificationManager.notify(notificationId, builder.build());

// When done, update the notification one more time to remove the progress bar
builder.setContentText("Download complete")
        .setProgress(0,0,false);
notificationManager.notify(notificationId, builder.build());
```

操作结束时，`progress` 应该等于 `max`。可以在操作完成后仍显示进度条，也可以将其移除。无论哪种情况，都请记得更新通知文本，显示操作已完成。如需移除进度条，请调用 `setProgress(0, 0, false)`。

如需显示不确定性进度条（不指示完成百分比的进度条），请调用 `setProgress(0, 0, true)`。结果会产生一个与上述进度条样式相同的指示器，区别是这个进度条是一个不指示完成情况的持续动画。在调用 `setProgress(0, 0, false)` 之前，进度动画会一直运行，调用后系统会更新通知以移除 Activity 指示器。

同时，请记得更改通知文本，表明操作已完成。

### 移除通知

除非发生以下情况之一，否则通知仍然可见：

- 用户关闭通知。
- 用户点击通知，且在创建通知时调用了 setAutoCancel()。
- 针对特定的通知 ID 调用了 cancel()。此方法还会删除当前通知。
- 调用了 cancelAll() 方法，该方法将移除之前发出的所有通知。
- 如果在创建通知时使用 setTimeoutAfter() 设置了超时，系统会在指定持续时间过后- 取消通知。如果需要，可以在指定的超时持续时间过去之前取消通知。

Copied via [android reference](https://developer.android.com/training/notify-user/build-notification#java)
