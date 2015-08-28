Giao tiếp với service sử dụng Pending Intent
=====

> Tham khảo
> - [Starting a Service](http://developer.android.com/guide/components/services.html#StartingAService)
> - [PendingIntent](http://developer.android.com/reference/android/app/PendingIntent.html)
> - [StackOverflow](http://stackoverflow.com/questions/6099364/how-to-use-pendingintent-to-communicate-from-a-service-to-a-client-activity)

# Giới thiệu

Việc sử dụng PendingIntent cho phép client gọi service để thực hiện một tác vụ nhưng service không cần biết chút thông tin gì về client.

> Client: thành phần của ứng dụng gọi service thông qua cơ chế intent để thực hiện một việc gì đó

**Lợi ích của việc sử dụng kĩ thuật này:**

1. Service không cần có thông tin gì về client
2. Chỉ cần thống nhất khuôn dạng dữ liệu mà service trả về cho client
3. Dễ dàng trao đổi dữ liệu giữa client và service trong cùng một app (dữ liệu không đi ra ngoài) và giữa client và service trong các app khác nhau (trao đổi dữ liệu liên ứng dụng)
4. Không cần dùng đến _bound_ service

Ở đây đề cập đến trường hợp client cũng là một service. Ngoài ra thì client cũng có thể là một activity. Tham khảo thêm vấn đề này tại [PendingIntent](http://developer.android.com/reference/android/app/PendingIntent.html)

# Ví dụ

Kịch bản ở đây là MainService gọi SecondaryService và nhận kết quả trả về.
Nếu thành công, kết quả (ở logcat) sẽ là:
```
MainService﹕ handleActionBaz: It WORKS!, It WORKS!
```

1. MainService

```
public class MainService extends IntentService {

    private static final Logger LOGGER = LoggerFactory.getLogger(MainService.class);

    // IntentService can perform, e.g. ACTION_FETCH_NEW_ITEMS
    private static final String ACTION_FOO = "com.example.p2p.demo.action.FOO";
    private static final String ACTION_BAZ = "com.example.p2p.demo.action.BAZ";
    private static final String EXTRA_PARAM1 = "com.example.p2p.demo.extra.PARAM1";
    private static final String EXTRA_PARAM2 = "com.example.p2p.demo.extra.PARAM2";

    /**
     * Starts this service to perform action Foo with the given parameters. If
     * the service is already performing a task this action will be queued.
     *
     * @see IntentService
     */
    public static void startActionFoo(Context context, String param1, String param2) {
        Intent intent = new Intent(context, MainService.class);
        intent.setAction(ACTION_FOO);
        intent.putExtra(EXTRA_PARAM1, param1);
        intent.putExtra(EXTRA_PARAM2, param2);
        context.startService(intent);
    }

    /**
     * Starts this service to perform action Baz with the given parameters. If
     * the service is already performing a task this action will be queued.
     *
     * @see IntentService
     */
    public static void startActionBaz(Context context, String param1, String param2) {
        Intent intent = new Intent(context, MainService.class);
        intent.setAction(ACTION_BAZ);
        intent.putExtra(EXTRA_PARAM1, param1);
        intent.putExtra(EXTRA_PARAM2, param2);
        context.startService(intent);
    }

    public MainService() {
        super("MainService");
    }

    @Override
    protected void onHandleIntent(Intent intent) {
        if (intent != null) {
            final String action = intent.getAction();
            if (ACTION_FOO.equals(action)) {
                final String param1 = intent.getStringExtra(EXTRA_PARAM1);
                final String param2 = intent.getStringExtra(EXTRA_PARAM2);
                handleActionFoo(param1, param2);
            } else if (ACTION_BAZ.equals(action)) {
                final String param1 = intent.getStringExtra(EXTRA_PARAM1);
                final String param2 = intent.getStringExtra(EXTRA_PARAM2);
                handleActionBaz(param1, param2);
            }
        }
    }

    /**
     * Handle action Foo in the provided background thread with the provided
     * parameters.
     */
    private void handleActionFoo(String param1, String param2) {
        LOGGER.info("handleActionFoo: " + param1 + ", " + param2);
        Intent mainServiceIntent = new Intent(this, MainService.class);
        mainServiceIntent.setAction(ACTION_BAZ);
        mainServiceIntent.putExtra(EXTRA_PARAM1, "It WORKS!");
        mainServiceIntent.putExtra(EXTRA_PARAM2, "It WORKS!");
        PendingIntent pi = PendingIntent.getService(this, 0, mainServiceIntent, 0);
        Bundle extras = new Bundle();
        extras.putParcelable(Constants.PENDING_INTENT, pi);

        Intent secondaryServiceIntent = new Intent(this, SecondaryService.class);
        secondaryServiceIntent.putExtras(extras);
        startService(secondaryServiceIntent);
    }

    /**
     * Handle action Baz in the provided background thread with the provided
     * parameters.
     */
    private void handleActionBaz(String param1, String param2) {
        LOGGER.info("handleActionBaz: " + param1 + ", " + param2);
    }
}
```

2. SecondaryService

```
public class SecondaryService extends IntentService {

    private static final Logger LOGGER = LoggerFactory.getLogger(SecondaryService.class);

    public SecondaryService() {
        super("SecondaryService");
    }

    @Override
    protected void onHandleIntent(Intent intent) {
        if (intent != null) {
            Bundle extras = intent.getExtras();
            PendingIntent replyPI = extras.getParcelable(Constants.PENDING_INTENT);
            Intent replyIntent = new Intent();
            replyIntent.putExtras(extras);
            try {
                if (replyPI != null) {
                    replyPI.send(this, 0, replyIntent);
                } else {
                    LOGGER.error("0x0001 No information about caller.");
                }
            } catch (PendingIntent.CanceledException e) {
                LOGGER.error("0x0002", e);
            }
        }
    }

}
```

3. Constants

```
public class Constants {
    public static final String PENDING_INTENT = "PendingIntent";
}
```
