## Problem 
The app attempts to make a network request even when no internet connection is available.

This leads to wasted power, unnecessary retries, and crashes or timeouts — a network misuse code smell.

### Where it happens ?

It happens inside your MainActivity.kt, when you try to fetch data without checking connectivity first.

The bad code assumes the device is always online.

---

## Original Code (Code Smell)
```cpp
package com.example.networkondemo

import android.os.Bundle
import android.widget.Button
import android.widget.TextView
import androidx.appcompat.app.AppCompatActivity
import java.net.HttpURLConnection
import java.net.URL

class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val tvStatus = findViewById<TextView>(R.id.tvStatus)
        val btnCheck = findViewById<Button>(R.id.btnCheck)

        btnCheck.setOnClickListener {
            tvStatus.text = "Fetching data..."

            //BAD: Tries to fetch from network without checking connectivity
            Thread {
                try {
                    val url = URL("https://httpbin.org/delay/3")
                    val connection = url.openConnection() as HttpURLConnection
                    connection.connect()
                    Thread.sleep(3000)
                    connection.disconnect()
                    runOnUiThread {
                        tvStatus.text = "Data fetched successfully"
                    }
                } catch (e: Exception) {
                    runOnUiThread {
                        tvStatus.text = "Error fetching data!"
                    }
                }
            }.start()
        }
    }
}


```



## Modified Code

```cpp
package com.example.networkondemo

import android.content.Context
import android.net.ConnectivityManager
import android.net.NetworkCapabilities
import android.os.Bundle
import android.widget.Button
import android.widget.TextView
import androidx.appcompat.app.AppCompatActivity
import java.net.HttpURLConnection
import java.net.URL

class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val tvStatus = findViewById<TextView>(R.id.tvStatus)
        val btnCheck = findViewById<Button>(R.id.btnCheck)

        btnCheck.setOnClickListener {
            if (!isNetworkAvailable()) {
                tvStatus.text = "No Internet Connection"
                return@setOnClickListener
            }

            tvStatus.text = "Fetching data..."
            Thread {
                try {
                    val url = URL("https://httpbin.org/delay/3")
                    val connection = url.openConnection() as HttpURLConnection
                    connection.connect()
                    Thread.sleep(3000)
                    connection.disconnect()
                    runOnUiThread {
                        tvStatus.text = "Data fetched successfully"
                    }
                } catch (e: Exception) {
                    runOnUiThread {
                        tvStatus.text = "Error fetching data"
                    }
                }
            }.start()
        }
    }

    private fun isNetworkAvailable(): Boolean {
        val connectivityManager =
            getSystemService(Context.CONNECTIVITY_SERVICE) as ConnectivityManager
        val network = connectivityManager.activeNetwork ?: return false
        val activeNetwork =
            connectivityManager.getNetworkCapabilities(network) ?: return false
        return activeNetwork.hasTransport(NetworkCapabilities.TRANSPORT_WIFI) ||
                activeNetwork.hasTransport(NetworkCapabilities.TRANSPORT_CELLULAR)
    }
}

```

