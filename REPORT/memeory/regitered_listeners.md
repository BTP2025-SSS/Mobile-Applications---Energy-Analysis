```java
class MainActivity : ComponentActivity() {

    // This BroadcastReceiver is an anonymous object, which acts like a non-static inner class.
    // It will hold an implicit reference to the MainActivity instance that created it.
    private val memoryLeakReceiver = object : BroadcastReceiver() {
        override fun onReceive(context: Context?, intent: Intent?) {
            // The simple fact that this object is registered and holds a reference
            // to MainActivity is what causes the leak.
            Log.d("LeakTest", "Broadcast received! Leaking context: ${this@MainActivity}")
        }
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            Text("Rotating this screen will cause a memory leak.")
        }

        // 1. We register the receiver with an IntentFilter.
        // The Android system now holds a strong reference to our `memoryLeakReceiver` object.
        val intentFilter = IntentFilter("com.example.myapplication.ACTION_TEST")
        //it is to liten from in rparticular action like network, booting etc

        // Note: For Android 13+ this requires an export flag.
        // This ensures the app runs so the leak can be demonstrated.
        val receiverFlags = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
            ContextCompat.RECEIVER_NOT_EXPORTED
        } else {
            0
        }
        ContextCompat.registerReceiver(this, memoryLeakReceiver, intentFilter, receiverFlags)
        // registred a reciver with the intent filter

        Log.d("LeakTest", "Receiver registered.")
    }

    override fun onDestroy() {
        super.onDestroy()
        Log.d("LeakTest", "onDestroy called, but receiver is NOT unregistered.")

        // 2. Because we do not call `unregisterReceiver(memoryLeakReceiver)` here,
        // the system keeps a reference to the receiver, which in turn keeps a reference
        // to the destroyed MainActivity, causing a memory leak.
    }

}
```
- Here tthere is no need of leakvcanary, the compiler wil l only detect the unregistyerd listeners. that causes leak.

```
Activity com.example.myapplication.MainActivity has leaked IntentReceiver com.example.myapplication.MainActivity$memoryLeakReceiver$1@f291df6 that was originally registered here. Are you missing a call to unregisterReceiver()?  android.app.IntentReceiverLeaked: Activity com.example.myapplication.MainActivity has leaked IntentReceiver com.example.myapplication.MainActivity$memoryLeakReceiver$1@f291df6 that was originally registered here. Are you missing a call to unregisterReceiver()?
                                                                                         	
```