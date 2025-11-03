## Problem 
The application performs a time-consuming network task on the main thread, causing the UI to freeze and become unresponsive.

This represents a Network/Threading Code Smell, where long operations should be moved off the main (UI) thread.

---

## Original Code (Code Smell)
```cpp
package com.example.networkondemo

import android.os.Bundle
import android.widget.Button
import android.widget.TextView
import androidx.appcompat.app.AppCompatActivity

class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val tvStatus = findViewById<TextView>(R.id.tvStatus)
        val btnCheck = findViewById<Button>(R.id.btnCheck)

        btnCheck.setOnClickListener {
            tvStatus.text = "Fetching data..."
            
            // BAD PRACTICE: Doing heavy work (10-sec delay) on main thread
            Thread.sleep(10000) // Simulates long-running task
            
            tvStatus.text = "Data fetched successfully!"
        }
    }
}

```

![alt text](<assets/smell1.jpg>)

## Modified Code

```cpp
package com.example.networkondemo

import android.os.Bundle
import android.widget.Button
import android.widget.TextView
import androidx.appcompat.app.AppCompatActivity

class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val tvStatus = findViewById<TextView>(R.id.tvStatus)
        val btnCheck = findViewById<Button>(R.id.btnCheck)

        btnCheck.setOnClickListener {
            tvStatus.text = "Fetching data..."

            // Correct: Run heavy task in a background thread
            Thread {
                Thread.sleep(10000) // Simulate 10 sec delay (e.g., network call)

                // Update UI back on main thread safely
                runOnUiThread {
                    tvStatus.text = "Data fetched successfully!"
                }
            }.start()
        }
    }
}

```

| Version          | Description                                         |
| ---------------- | --------------------------------------------------- |
| Code Smell | Runs network-like task on main thread → UI freezes. |
| Optimised  | Uses background thread → UI stays responsive.       |
