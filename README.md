dependencies {
    // Firebase
    implementation platform('com.google.firebase:firebase-bom:32.3.1')
    implementation 'com.google.firebase:firebase-auth-ktx'
    implementation 'com.google.firebase:firebase-firestore-ktx'

    // UI & Lifecycle
    implementation 'androidx.core:core-ktx:1.12.0'
    implementation 'androidx.appcompat:appcompat:1.6.1'
    implementation 'com.google.android.material:material:1.10.0'
    implementation 'androidx.constraintlayout:constraintlayout:2.1.4'
}<manifest xmlns:android="http://schemas.android.com/apk/res/xml/android"
    package="com.example.zoric">

    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.RECORD_AUDIO" />

    <application ... >
        <!-- Activities -->
    </application>
</manifest>
users/{userId}/paired_device_id: Stores the ID of the linked device.
devices/{deviceId}/: Stores battery_level, is_online, and owner_id.
<style name="Theme.Zoric" parent="Theme.MaterialComponents.DayNight.NoActionBar">
    <item name="colorPrimary">#BB86FC</item>
    <item name="android:windowBackground">#121212</item>
    <item name="colorSurface">#1E1E1E</item>
</style>class MainActivity : AppCompatActivity() {

    private lateinit var auth: FirebaseAuth
    private lateinit var db: FirebaseFirestore
    private var isRecording = false

    // UI Elements
    private lateinit var statusDot: View
    private lateinit var batteryText: TextView
    private lateinit var micButton: ExtendedFloatingActionButton
    private lateinit var stopButton: Button
    private lateinit var recordingIndicator: LinearLayout

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        auth = Firebase.auth
        db = Firebase.firestore

        setupUI()
        observePairedDevice()
        startBatteryReporting()
    }

    private fun setupUI() {
        micButton = findViewById(R.id.btn_mic)
        stopButton = findViewById(R.id.btn_stop)
        recordingIndicator = findViewById(R.id.layout_recording_indicator)
        
        micButton.setOnClickListener {
            checkPermissionAndStartMic()
        }

        stopButton.setOnClickListener {
            stopMicrophone()
        }
    }

    private fun checkPermissionAndStartMic() {
        if (ContextCompat.checkSelfPermission(this, Manifest.permission.RECORD_AUDIO) 
            != PackageManager.PERMISSION_GRANTED) {
            ActivityCompat.requestPermissions(this, arrayOf(Manifest.permission.RECORD_AUDIO), 101)
        } else {
            startMicrophone()
        }
    }

    private fun startMicrophone() {
        // High visibility UI change
        isRecording = true
        micButton.visibility = View.GONE
        stopButton.visibility = View.VISIBLE
        recordingIndicator.visibility = View.VISIBLE
        
        Toast.makeText(this, "Microphone Active", Toast.LENGTH_SHORT).show()
        // Initialize AudioRecord or MediaRecorder here
    }

    private fun stopMicrophone() {
        isRecording = false
        micButton.visibility = View.VISIBLE
        stopButton.visibility = View.GONE
        recordingIndicator.visibility = View.GONE
        
        Toast.makeText(this, "Microphone Stopped", Toast.LENGTH_SHORT).show()
        // Release recorder resources here
    }

    private fun observePairedDevice() {
        val currentUserId = auth.currentUser?.uid ?: return
        
        // Listen for device updates in Real-time
        db.collection("devices").document(currentUserId)
            .addSnapshotListener { snapshot, _ ->
                if (snapshot != null && snapshot.exists()) {
                    val isOnline = snapshot.getBoolean("is_online") ?: false
                    val battery = snapshot.getLong("battery_level") ?: 0
                    
                    batteryText.text = "$battery%"
                    statusDot.setBackgroundResource(
                        if (isOnline) R.drawable.dot_green else R.drawable.dot_red
                    )
                }
            }
    }

    private fun startBatteryReporting() {
        // Periodically update the Firestore with this device's battery
        val bm = getSystemService(BATTERY_SERVICE) as BatteryManager
        val level = bm.getIntProperty(BatteryManager.BATTERY_PROPERTY_CAPACITY)
        
        val deviceData = hashMapOf(
            "battery_level" to level,
            "is_online" to true,
            "last_updated" to FieldValue.serverTimestamp()
        )
        
        auth.currentUser?.uid?.let { uid ->
            db.collection("devices").document(uid).set(deviceData, SetOptions.merge())
        }
    }

    override fun onDestroy() {
        super.onDestroy()
        // Set offline status on exit
        auth.currentUser?.uid?.let { uid ->
            db.collection("devices").document(uid).update("is_online", false)
        }
    }
}Generate Code: Device A generates a random 6-digit code stored in Firestore under pending_pairs.
Enter Code: User B enters Device A's code.
Validation: The app checks if the code is valid. If yes, it updates Device A's document with authorized_user: UserB_ID.
Mutual Link: Only after Device A's owner sees a popup and clicks "Approve" is the connection finalized.elativeLayout xmlns:android="http://schemas.android.com/apk/res/xml/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="#121212"
    android:padding="24dp">

    <TextView
        android:id="@+id/app_title"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Zoric"
        android:textColor="#BB86FC"
        android:textSize="32sp"
        android:textStyle="bold" />

    <!-- Status Card -->
    <LinearLayout
        android:id="@+id/status_card"
        android:layout_below="@id/app_title"
        android:layout_marginTop="30dp"
        android:background="#1E1E1E"
        android:padding="20dp"
        android:orientation="vertical">

        <TextView
            android:id="@+id/battery_text"
            android:text="Battery: --%"
            android:textColor="#FFFFFF" />

        <View
            android:id="@+id/status_dot"
            android:layout_width="12dp"
            android:layout_height="12dp"
            android:background="@drawable/dot_red" />
    </LinearLayout>

    <!-- Recording Indicator (Hidden by default) -->
    <LinearLayout
        android:id="@+id/layout_recording_indicator"
        android:layout_centerInParent="true"
        android:visibility="gone"
        android:orientation="horizontal">
        <ProgressBar android:layout_width="20dp" android:layout_height="20dp" />
        <TextView
            android:text="Microphone ON"
            android:textColor="#FF4444"
            android:layout_marginStart="10dp"/>
    </LinearLayout>

    <com.google.android.material.floatingactionbutton.ExtendedFloatingActionButton
        android:id="@+id/btn_mic"
        android:layout_alignParentBottom="true"
        android:layout_centerHorizontal="true"
        android:text="Start Listening"
        app:icon="@drawable/ic_mic" />

    <Button
        android:id="@+id/btn_stop"
        android:layout_alignParentBottom="true"
        android:layout_centerHorizontal="true"
        android:text="STOP"
        android:visibility="gone"
        android:backgroundTint="#CF6679" />

</RelativeLayout>
