# No Response Caching (Redundant Network Calls)

## Problem
The app makes a network request every time the user performs an action, even if the data has not changed.
This causes unnecessary bandwidth use, battery drain, and slow UI, especially when the same data is fetched repeatedly.

It wastes resources because the app ignores local caching mechanisms.

## Original Code (Code Smell)

```
package com.example.networkondemo

import android.os.Bundle
import android.widget.Button
import android.widget.TextView
import androidx.appcompat.app.AppCompatActivity
import java.net.HttpURLConnection
import java.net.URL
import java.io.BufferedReader
import java.io.InputStreamReader

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

                    // Code smell: Always calls API even if same data was fetched before
                    connection.connect()

                    val reader = BufferedReader(InputStreamReader(connection.inputStream))
                    val response = reader.readText()

                    runOnUiThread {
                        tvStatus.text = "Response: ${response.take(50)}..."
                    }

                    reader.close()
                    connection.disconnect()
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

![alt text](<assets/reqSmell.png>)

## Modified Code

```
package com.example.networkondemo

import android.os.Bundle
import android.widget.Button
import android.widget.TextView
import androidx.appcompat.app.AppCompatActivity
import java.net.HttpURLConnection
import java.net.URL
import java.io.BufferedReader
import java.io.InputStreamReader

class MainActivity : AppCompatActivity() {

    private var cachedResponse: String? = null // cache to store previous data

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val tvStatus = findViewById<TextView>(R.id.tvStatus)
        val btnFetch = findViewById<Button>(R.id.btnCheck)

        btnFetch.setOnClickListener {
            tvStatus.text = "Checking cache..."
            if (cachedResponse != null) {
                // Use cached data if available
                tvStatus.text = "Loaded from cache: ${cachedResponse!!.take(50)}..."
            } else {
                // Fetch from network only once
                tvStatus.text = "Fetching from network..."
                Thread {
                    try {
                        val url = URL("https://httpbin.org/get")
                        val connection = url.openConnection() as HttpURLConnection
                        connection.requestMethod = "GET"
                        connection.connect()

                        val reader = BufferedReader(InputStreamReader(connection.inputStream))
                        val response = reader.readText()
                        cachedResponse = response // store response in cache

                        runOnUiThread {
                            tvStatus.text = "Fetched: ${response.take(50)}..."
                        }

                        reader.close()
                        connection.disconnect()
                    } catch (e: Exception) {
                        runOnUiThread {
                            tvStatus.text = "Error: ${e.message}"
                        }
                    }
                }.start()
            }
        }
    }
}

```

| Aspect             | Before Fix                   | After Fix                          |
| ------------------ | ---------------------------- | ---------------------------------- |
| Network requests   | Every click triggers network | First click only (next from cache) |
| Data speed         | Slow (depends on internet)   | Fast (instant cache response)      |
| Battery/data usage | High                         | Low                                |
| User experience    | Laggy and redundant          | Smooth and responsive              |
