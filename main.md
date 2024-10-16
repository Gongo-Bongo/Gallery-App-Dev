Here's a full step-by-step guide for creating an Android app using Kotlin and Jetpack Compose to display a gallery of images and videos from a specific GitHub repository.

### 1. **Create a New Android Studio Project**
- Open Android Studio.
- Select **New Project** > **Empty Compose Activity**.
- Set the **Language** to Kotlin, and select the **Minimum API Level** (e.g., API 21 or above).
- Click **Finish** to create the project.

---

### 2. **Add Dependencies in `build.gradle`**
In the `build.gradle (Module: app)` file, add the following dependencies for Jetpack Compose, Retrofit (for API requests), Coil (for loading images), and ExoPlayer (for video playback):

```groovy
dependencies {
    // Jetpack Compose
    implementation("androidx.compose.ui:ui:1.4.3")
    implementation("androidx.compose.material:material:1.4.3")
    implementation("androidx.compose.ui:ui-tooling-preview:1.4.3")
    implementation("androidx.lifecycle:lifecycle-runtime-compose:2.6.1")

    // Retrofit for network calls
    implementation("com.squareup.retrofit2:retrofit:2.9.0")
    implementation("com.squareup.retrofit2:converter-gson:2.9.0")

    // Coil for image loading
    implementation("io.coil-kt:coil-compose:2.4.0")

    // ExoPlayer for video playback
    implementation("com.google.android.exoplayer:exoplayer:2.18.5")

    // Coroutine for async tasks
    implementation "org.jetbrains.kotlinx:kotlinx-coroutines-android:1.6.4"
}
```

Click **Sync Now** when prompted to sync your project with the new dependencies.

---

### 3. **Create the Data Model (`RepoFile.kt`)**
Create a new Kotlin file named `RepoFile.kt` inside the `app/src/main/java/com/yourappname` directory.

```kotlin
data class RepoFile(
    val name: String,
    val path: String,
    val type: String,  // "file" or "dir"
    val download_url: String?
)
```

---

### 4. **Create the GitHub API Service (`GitHubApiService.kt`)**
Create a new Kotlin file named `GitHubApiService.kt` inside the same directory:

```kotlin
import retrofit2.http.GET
import retrofit2.http.Path

interface GitHubApiService {
    @GET("repos/{owner}/{repo}/contents/{path}")
    suspend fun getRepoContents(
        @Path("owner") owner: String,
        @Path("repo") repo: String,
        @Path("path") path: String = ""
    ): List<RepoFile>
}
```

---

### 5. **Set Up Retrofit Instance (`RetrofitInstance.kt`)**
Create another Kotlin file named `RetrofitInstance.kt`:

```kotlin
import retrofit2.Retrofit
import retrofit2.converter.gson.GsonConverterFactory

object RetrofitInstance {
    private val retrofit by lazy {
        Retrofit.Builder()
            .baseUrl("https://api.github.com/")
            .addConverterFactory(GsonConverterFactory.create())
            .build()
    }

    val api: GitHubApiService by lazy {
        retrofit.create(GitHubApiService::class.java)
    }
}
```

---

### 6. **Create ViewModel (`RepoViewModel.kt`)**
Create a ViewModel file named `RepoViewModel.kt`. This ViewModel will fetch the repository contents using Retrofit.

```kotlin
import androidx.compose.runtime.*
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.launch

class RepoViewModel : ViewModel() {
    var repoFiles by mutableStateOf<List<RepoFile>>(emptyList())
        private set

    fun fetchRepoContents(owner: String, repo: String, path: String = "") {
        viewModelScope.launch {
            try {
                val response = RetrofitInstance.api.getRepoContents(owner, repo, path)
                repoFiles = response.filter {
                    it.type == "file" && 
                    (it.name.endsWith(".jpg") || it.name.endsWith(".png") || it.name.endsWith(".mp4"))
                }
            } catch (e: Exception) {
                // Handle error
            }
        }
    }
}
```

---

### 7. **Create UI Components**

#### **7.1 Create the Video Player Component (`VideoPlayer.kt`)**
Create a new Kotlin file named `VideoPlayer.kt` to display videos using ExoPlayer.

```kotlin
import android.content.Context
import androidx.compose.foundation.layout.*
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.viewinterop.AndroidView
import com.google.android.exoplayer2.ExoPlayer
import com.google.android.exoplayer2.MediaItem
import com.google.android.exoplayer2.ui.PlayerView

@Composable
fun VideoPlayer(url: String) {
    val context = LocalContext.current
    val exoPlayer = rememberExoPlayer(context, url)

    AndroidView(
        factory = { PlayerView(context).apply { player = exoPlayer } },
        modifier = Modifier
            .fillMaxWidth()
            .height(200.dp)
    )
}

@Composable
fun rememberExoPlayer(context: Context, url: String): ExoPlayer {
    return remember {
        ExoPlayer.Builder(context).build().apply {
            setMediaItem(MediaItem.fromUri(url))
            prepare()
            playWhenReady = true
        }
    }
}
```

---

#### **7.2 Create the Main UI (`RepoGalleryScreen.kt`)**
Create another file `RepoGalleryScreen.kt` where you define the screen to show the images and videos.

```kotlin
import androidx.compose.foundation.Image
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import coil.compose.rememberImagePainter

@Composable
fun RepoGalleryScreen(viewModel: RepoViewModel) {
    val repoFiles = viewModel.repoFiles

    LazyColumn(
        modifier = Modifier.fillMaxSize(),
        contentPadding = PaddingValues(16.dp)
    ) {
        items(repoFiles) { file ->
            if (file.name.endsWith(".jpg") || file.name.endsWith(".png")) {
                Image(
                    painter = rememberImagePainter(file.download_url),
                    contentDescription = file.name,
                    modifier = Modifier
                        .fillMaxWidth()
                        .height(200.dp)
                )
            } else if (file.name.endsWith(".mp4")) {
                VideoPlayer(url = file.download_url ?: "")
            }

            Spacer(modifier = Modifier.height(16.dp))
        }
    }
}
```

---

### 8. **Set Up MainActivity (`MainActivity.kt`)**
In your `MainActivity.kt` file, set up the ViewModel and call the `RepoGalleryScreen` to show the images and videos.

```kotlin
package com.example.instasexies

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.viewModels
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.padding
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Scaffold
import androidx.compose.material3.Surface
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.tooling.preview.Preview
import com.example.instasexies.ui.theme.InstaSexiesTheme

class MainActivity : ComponentActivity() {
    private val viewModel: RepoViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            InstaSexiesTheme {
                Scaffold { paddingValues ->
                    Surface(
                        modifier = Modifier
                            .fillMaxSize()
                            .padding(paddingValues),
                        color = MaterialTheme.colorScheme.background
                    ) {
                        RepoGalleryScreen(viewModel)
                    }
                }
            }
        }

        // Fetch the contents from the GitHub repository
        viewModel.fetchRepoContents(
            owner = "gitporn69",   // Change this
            repo = "instaserver",  // Change this
            path = "path/to/media" // Change this if necessary
        )
    }
}

@Composable
fun Greeting(name: String, modifier: Modifier = Modifier) {
    Text(
        text = "Hello $name!",
        modifier = modifier
    )
}

@Preview(showBackground = true)
@Composable
fun GreetingPreview() {
    InstaSexiesTheme {
        Greeting("Android")
    }
}

```

---

### 9. **Run the App**

- Run the app on an Android device or emulator.
- The app will load the images and videos from the specified GitHub repository and display them in a list.
- Adjust the repository details (`owner`, `repo`, and `path`) as per your repository setup.

### Key Notes:
- **Error Handling**: You can extend the app to handle API errors, network failures, or loading states (e.g., showing a loading spinner while the data is fetched).
- **Video Caching**: Consider adding video caching for better performance.

### Extra Debugging:

Your `MainActivity.kt` file looks mostly good, but there are a few things to review or improve. Here's a breakdown:

### 1. **Material Theme Usage**
You're mixing **MaterialTheme** from **Material 2** with **Material 3** (you imported both in `MainActivity`). Since you're likely using Jetpack Compose Material 3, it would be better to use `MaterialTheme` from the Material 3 library instead.

### 2. **Internet Permission**
Make sure you have added internet permission in your `AndroidManifest.xml`. Without this, the app won't be able to fetch data from the GitHub repository.

```xml
<uses-permission android:name="android.permission.INTERNET"/>
```

### 3. **Fetching Content from GitHub**:
In `onCreate`, you're calling `viewModel.fetchRepoContents(...)` right after setting the content. This is fine, but ensure that the ViewModel and the GitHub API fetching mechanism are correctly implemented and that they handle potential exceptions, like network errors.

### 4. **Jetpack Compose Scaffold (Optional Enhancement)**
It might be useful to wrap your content in a `Scaffold` if you plan to add things like top bars or floating action buttons. Here's an updated version of your `MainActivity` with `Scaffold` and `Material 3` improvements:

### Revised `MainActivity.kt`

```kotlin
package com.example.instasexies

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.viewModels
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.padding
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Scaffold
import androidx.compose.material3.Surface
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.tooling.preview.Preview
import com.example.instasexies.ui.theme.InstaSexiesTheme

class MainActivity : ComponentActivity() {
    private val viewModel: RepoViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            InstaSexiesTheme {
                Scaffold { paddingValues ->
                    Surface(
                        modifier = Modifier
                            .fillMaxSize()
                            .padding(paddingValues),
                        color = MaterialTheme.colorScheme.background
                    ) {
                        RepoGalleryScreen(viewModel)
                    }
                }
            }
        }

        // Fetch the contents from the GitHub repository
        viewModel.fetchRepoContents(
            owner = "gitporn69",   // Change this
            repo = "instaserver",  // Change this
            path = "path/to/media" // Change this if necessary
        )
    }
}

@Composable
fun Greeting(name: String, modifier: Modifier = Modifier) {
    Text(
        text = "Hello $name!",
        modifier = modifier
    )
}

@Preview(showBackground = true)
@Composable
fun GreetingPreview() {
    InstaSexiesTheme {
        Greeting("Android")
    }
}
```

### Changes Made:
1. **Material 3 Setup**: `MaterialTheme` and `Surface` are now using Material 3 (as per `MaterialTheme.colorScheme`).
2. **Scaffold Added**: The `Scaffold` component is added, which provides padding for the content and is useful for future UI additions like toolbars or bottom navigation.
3. **Surface Background Color**: The background is set to the Material 3 theme background color using `MaterialTheme.colorScheme.background`.

### 5. **Troubleshooting: Check the ViewModel**

Make sure your `RepoViewModel` is correctly implemented and handles the following:
- **Coroutines for network calls** (no blocking of the main thread).
- **Error handling** (e.g., if the network request fails).
- **LiveData/StateFlow** to observe changes and update the UI reactively.

### 6. **Error Handling in the ViewModel**
If your GitHub API calls fail, your app might crash or render nothing. Ensure you're catching exceptions and showing some error UI in case of a failure.

For example, inside your `RepoViewModel`, handle exceptions:

```kotlin
fun fetchRepoContents(owner: String, repo: String, path: String) {
    viewModelScope.launch {
        try {
            val response = apiService.getRepoContents(owner, repo, path)
            _repoFiles.value = response
        } catch (e: Exception) {
            _errorState.value = true
            // Log or handle the error
        }
    }
}
```

You could then update your UI based on `_errorState` to show a message when an error occurs.

---

Let me know if the updated `MainActivity.kt` works for you or if the app is still crashing!