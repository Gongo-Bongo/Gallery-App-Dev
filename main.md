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
    implementation 'com.squareup.retrofit2:retrofit:2.9.0'
    implementation 'com.squareup.retrofit2:converter-gson:2.9.0'

    // Coil for image loading
    implementation("io.coil-kt:coil-compose:2.4.0")

    // ExoPlayer for video playback
    implementation 'com.google.android.exoplayer:exoplayer:2.18.5'

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
import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.viewModels
import androidx.compose.material.MaterialTheme
import androidx.compose.material.Surface

class MainActivity : ComponentActivity() {
    private val viewModel: RepoViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MaterialTheme {
                Surface {
                    RepoGalleryScreen(viewModel)
                }
            }
        }

        // Fetch the contents from the GitHub repository
        viewModel.fetchRepoContents(
            owner = "YourGitHubUsername", // Change this
            repo = "YourRepoName",        // Change this
            path = "path/to/media"        // Change this if necessary
        )
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

Let me know if you need any additional details or help setting up your project!