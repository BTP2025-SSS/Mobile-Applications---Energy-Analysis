## Not Closing Network Connections Properly

### Problem 
Many Android apps open HttpURLConnection or network streams but forget to close them after usage.
This is a network and resource management code smell, leading to:

1. Resource leaks (sockets remain open)

2. Increased battery and network usage

3. Too many open files errors

4. Crashes on some devices after multiple requests

---

## Original Code (Code Smell)

```
package com.example.networkondemo

import android.os.Bundle
import android.widget.Button
import android.widget.TextView
import androidx.appcompat.app.AppCompatActivity
import java.io.BufferedReader
import java.io.InputStreamReader
import java.net.HttpURLConnection
import java.net.URL

class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val tvStatus = findViewById<TextView>(R.id.tvStatus)
        val btnFetch = findViewById<Button>(R.id.btnCheck)

        btnFetch.setOnClickListener {
            tvStatus.text = "Fetching data..."
            Thread {
                try {
                    val url = URL("https://httpbin.org/get")
                    val connection = url.openConnection() as HttpURLConnection
                    connection.requestMethod = "GET"
                    connection.connect()

                    val reader = BufferedReader(InputStreamReader(connection.inputStream))
                    val response = StringBuilder()
                    var line: String?

                    while (reader.readLine().also { line = it } != null) {
                        response.append(line)
                    }

                    // Code smell: connection and reader not closed
                    // reader.close() and connection.disconnect() missing

                    runOnUiThread {
                        tvStatus.text = "Response: ${response.toString().take(50)}..."
                    }

                } catch (e: Exception) {
                    runOnUiThread {
                        tvStatus.text = "Error: ${e.message}"
                    }
                }
            }.start()
        }
    }
}
```

Works fine initially, but after multiple network requests app starts slowing down , It will show us too many files opened, and it may crash due to socket leak , it consumes network and battery leading to hang the device (happened multiple times while testing (screen freezes doesnt respond))..


## Modified Code
```
package com.example.networkondemo

import android.os.Bundle
import android.widget.Button
import android.widget.TextView
import androidx.appcompat.app.AppCompatActivity
import java.io.BufferedReader
import java.io.InputStreamReader
import java.net.HttpURLConnection
import java.net.URL

class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val tvStatus = findViewById<TextView>(R.id.tvStatus)
        val btnFetch = findViewById<Button>(R.id.btnCheck)

        btnFetch.setOnClickListener {
            tvStatus.text = "Fetching data..."
            Thread {
                var connection: HttpURLConnection? = null
                var reader: BufferedReader? = null
                try {
                    val url = URL("https://httpbin.org/get")
                    connection = url.openConnection() as HttpURLConnection
                    connection.requestMethod = "GET"
                    connection.connect()

                    reader = BufferedReader(InputStreamReader(connection.inputStream))
                    val response = StringBuilder()
                    var line: String?

                    while (reader.readLine().also { line = it } != null) {
                        response.append(line)
                    }

                    runOnUiThread {
                        tvStatus.text = "Response: ${response.toString().take(50)}..."
                    }

                } catch (e: Exception) {
                    runOnUiThread {
                        tvStatus.text = "Error: ${e.message}"
                    }
                } finally {
                    // Properly closing resources
                    reader?.close()
                    connection?.disconnect()
                }
            }.start()
        }
    }
}

```

| Aspect         | Before Fix                                  | After Fix                       |
| -------------- | ------------------------------------------- | ------------------------------- |
| Resource usage | Memory & socket leaks accumulate            | Resources released properly     |
| Performance    | App slows after multiple requests           | Smooth even after many requests |
| Battery drain  | Increases due to hanging sockets            | Reduced battery usage           |
| Logcat         | May show “Resource never released” warnings | No warnings                     |


| Test Case                  | Description               | Expected Result      | Observation                       |
| -------------------------- | ------------------------- | -------------------- | --------------------------------- |
| Without closing connection | Make 10–15 API calls      | App slows or crashes | “Resource never released” appears |
| With proper closing        | Make same 10–15 API calls | Works smoothly       | No errors in Logcat               |
