## Problem 
In many Android applications, developers directly write API URLs or endpoints inside activities or fragments (for example, "https://api.example.com/getData").
This practice is called hardcoding and is a network-related code smell.

It becomes a problem when:

The URL or endpoint changes it must be updated in multiple files manually.

There are different environments (development, testing, production) needing different URLs.

It violates the Single Source of Truth and Separation of Concerns design principles.

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

        val statusText = findViewById<TextView>(R.id.tvStatus)
        val checkButton = findViewById<Button>(R.id.btnCheck)

        checkButton.setOnClickListener {
            statusText.text = "Fetching data..."
            Thread {
                try {
                    // Hardcoded URL — Code Smell
                    val url = URL("https://api.example.com/getData")
                    val connection = url.openConnection() as HttpURLConnection
                    connection.connect()
                    val responseCode = connection.responseCode
                    runOnUiThread {
                        statusText.text = "Response code: $responseCode"
                    }
                } catch (e: Exception) {
                    runOnUiThread {
                        statusText.text = "Error: ${e.message}"
                    }
                }
            }.start()
        }
    }
}



```


## Modified Code

Here we need to maintain 2 files..

**ApiConstants.kt**

```cpp
package com.example.networkondemo.utils

object ApiConstants {
    // Define all endpoints in one place
    const val BASE_URL = "https://api.example.com/"
    const val DATA_ENDPOINT = "getData"
}

```

1. Behavior (When Code Smell is Present)

2. If the API base URL changes, the app will fail to connect (you must manually edit code and rebuild).

3. Testing or staging environments can’t be switched easily.

4. In larger apps with 10+ API calls, maintenance becomes error-prone.

5. Hardcoded URLs reduce reusability and configurability of code.

6. Developers may accidentally expose production endpoints in public GitHub repositories


```cpp
package com.example.networkondemo

import android.os.Bundle
import android.widget.Button
import android.widget.TextView
import androidx.appcompat.app.AppCompatActivity
import com.example.networkondemo.utils.ApiConstants
import java.net.HttpURLConnection
import java.net.URL

class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val statusText = findViewById<TextView>(R.id.tvStatus)
        val checkButton = findViewById<Button>(R.id.btnCheck)

        checkButton.setOnClickListener {
            statusText.text = "Fetching data..."
            Thread {
                try {
                    // URL constructed from constants
                    val url = URL(ApiConstants.BASE_URL + ApiConstants.DATA_ENDPOINT)
                    val connection = url.openConnection() as HttpURLConnection
                    connection.connect()
                    val responseCode = connection.responseCode
                    runOnUiThread {
                        statusText.text = "Response code: $responseCode"
                    }
                } catch (e: Exception) {
                    runOnUiThread {
                        statusText.text = "Error: ${e.message}"
                    }
                }
            }.start()
        }
    }
}


```

| Test Case           | Description                            | Expected Result           | Observation                             |
| ------------------- | -------------------------------------- | ------------------------- | --------------------------------------- |
| Using hardcoded URL | Change API endpoint manually           | App fails to connect      | UnknownHostException or 404 in Logcat |
| Using constant URL  | Change BASE_URL once in constants file | App connects successfully | Shows response normally                 |

