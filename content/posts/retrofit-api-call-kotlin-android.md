---
title: "Panduan Retrofit: Implementasi API Call dengan Kotlin di Android"
date: 2024-11-29
categories: ["android"]
tags: ["kotlin", "retrofit", "api", "networking"]
featured_image: "images/posts/comp.jpg"
draft: false
---

Retrofit adalah library HTTP client terpopuler untuk Android. Pelajari cara implementasi yang benar dengan Kotlin dan best practices.

## Setup Dependencies

Tambahkan dependencies di `build.gradle (Module: app)`:

```kotlin
dependencies {
    implementation 'com.squareup.retrofit2:retrofit:2.9.0'
    implementation 'com.squareup.retrofit2:converter-gson:2.9.0'
    implementation 'com.squareup.okhttp3:logging-interceptor:4.11.0'
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3'
}
```

## Data Models

Buat data class untuk response API:

```kotlin
data class User(
    @SerializedName("id")
    val id: Int,
    @SerializedName("name")
    val name: String,
    @SerializedName("email")
    val email: String,
    @SerializedName("phone")
    val phone: String,
    @SerializedName("website")
    val website: String
)

data class Post(
    @SerializedName("userId")
    val userId: Int,
    @SerializedName("id")
    val id: Int,
    @SerializedName("title")
    val title: String,
    @SerializedName("body")
    val body: String
)

data class ApiResponse<T>(
    @SerializedName("status")
    val status: String,
    @SerializedName("message")
    val message: String,
    @SerializedName("data")
    val data: T?
)
```

## API Interface

Definisikan endpoints API:

```kotlin
interface ApiService {
    
    @GET("users")
    suspend fun getUsers(): Response<List<User>>
    
    @GET("users/{id}")
    suspend fun getUserById(@Path("id") userId: Int): Response<User>
    
    @GET("posts")
    suspend fun getPosts(): Response<List<Post>>
    
    @POST("users")
    suspend fun createUser(@Body user: User): Response<User>
    
    @PUT("users/{id}")
    suspend fun updateUser(
        @Path("id") userId: Int,
        @Body user: User
    ): Response<User>
    
    @DELETE("users/{id}")
    suspend fun deleteUser(@Path("id") userId: Int): Response<Unit>
    
    @GET("posts")
    suspend fun getPostsWithQuery(
        @Query("userId") userId: Int?,
        @Query("page") page: Int = 1,
        @Query("limit") limit: Int = 10
    ): Response<List<Post>>
}
```

## Retrofit Client Setup

Buat Retrofit client dengan interceptor:

```kotlin
object RetrofitClient {
    
    private const val BASE_URL = "https://jsonplaceholder.typicode.com/"
    
    private val loggingInterceptor = HttpLoggingInterceptor().apply {
        level = if (BuildConfig.DEBUG) {
            HttpLoggingInterceptor.Level.BODY
        } else {
            HttpLoggingInterceptor.Level.NONE
        }
    }
    
    private val httpClient = OkHttpClient.Builder()
        .addInterceptor(loggingInterceptor)
        .addInterceptor { chain ->
            val originalRequest = chain.request()
            val newRequest = originalRequest.newBuilder()
                .addHeader("Content-Type", "application/json")
                .addHeader("Accept", "application/json")
                .build()
            chain.proceed(newRequest)
        }
        .connectTimeout(30, TimeUnit.SECONDS)
        .readTimeout(30, TimeUnit.SECONDS)
        .writeTimeout(30, TimeUnit.SECONDS)
        .build()
    
    val instance: ApiService by lazy {
        Retrofit.Builder()
            .baseUrl(BASE_URL)
            .client(httpClient)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
            .create(ApiService::class.java)
    }
}
```

## Repository Pattern

Implementasi repository untuk data layer:

```kotlin
class UserRepository {
    
    private val apiService = RetrofitClient.instance
    
    suspend fun getUsers(): Resource<List<User>> {
        return try {
            val response = apiService.getUsers()
            if (response.isSuccessful) {
                Resource.Success(response.body() ?: emptyList())
            } else {
                Resource.Error("Error: ${response.code()} ${response.message()}")
            }
        } catch (e: Exception) {
            Resource.Error("Network error: ${e.message}")
        }
    }
    
    suspend fun getUserById(userId: Int): Resource<User> {
        return try {
            val response = apiService.getUserById(userId)
            if (response.isSuccessful) {
                response.body()?.let { user ->
                    Resource.Success(user)
                } ?: Resource.Error("User not found")
            } else {
                Resource.Error("Error: ${response.code()} ${response.message()}")
            }
        } catch (e: Exception) {
            Resource.Error("Network error: ${e.message}")
        }
    }
    
    suspend fun createUser(user: User): Resource<User> {
        return try {
            val response = apiService.createUser(user)
            if (response.isSuccessful) {
                Resource.Success(response.body()!!)
            } else {
                Resource.Error("Failed to create user")
            }
        } catch (e: Exception) {
            Resource.Error("Network error: ${e.message}")
        }
    }
}

sealed class Resource<T> {
    data class Success<T>(val data: T) : Resource<T>()
    data class Error<T>(val message: String) : Resource<T>()
    data class Loading<T>(val data: T? = null) : Resource<T>()
}
```

## ViewModel Implementation

Implementasi ViewModel dengan Coroutines:

```kotlin
class UserViewModel : ViewModel() {
    
    private val repository = UserRepository()
    
    private val _users = MutableLiveData<Resource<List<User>>>()
    val users: LiveData<Resource<List<User>>> = _users
    
    private val _selectedUser = MutableLiveData<Resource<User>>()
    val selectedUser: LiveData<Resource<User>> = _selectedUser
    
    fun fetchUsers() {
        _users.value = Resource.Loading()
        viewModelScope.launch {
            try {
                val result = repository.getUsers()
                _users.value = result
            } catch (e: Exception) {
                _users.value = Resource.Error("Failed to fetch users: ${e.message}")
            }
        }
    }
    
    fun fetchUserById(userId: Int) {
        _selectedUser.value = Resource.Loading()
        viewModelScope.launch {
            try {
                val result = repository.getUserById(userId)
                _selectedUser.value = result
            } catch (e: Exception) {
                _selectedUser.value = Resource.Error("Failed to fetch user: ${e.message}")
            }
        }
    }
    
    fun createUser(user: User) {
        viewModelScope.launch {
            try {
                val result = repository.createUser(user)
                when (result) {
                    is Resource.Success -> {
                        // Refresh users list
                        fetchUsers()
                    }
                    is Resource.Error -> {
                        // Handle error
                        Log.e("UserViewModel", "Error creating user: ${result.message}")
                    }
                    else -> {}
                }
            } catch (e: Exception) {
                Log.e("UserViewModel", "Exception creating user: ${e.message}")
            }
        }
    }
}
```

## Activity Implementation

Menggunakan API call di Activity:

```kotlin
class MainActivity : AppCompatActivity() {
    
    private lateinit var binding: ActivityMainBinding
    private lateinit var userAdapter: UserAdapter
    private lateinit var viewModel: UserViewModel
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)
        
        viewModel = ViewModelProvider(this)[UserViewModel::class.java]
        
        setupRecyclerView()
        observeViewModel()
        
        // Fetch users on app start
        viewModel.fetchUsers()
        
        // Refresh on swipe
        binding.swipeRefreshLayout.setOnRefreshListener {
            viewModel.fetchUsers()
        }
    }
    
    private fun observeViewModel() {
        viewModel.users.observe(this) { resource ->
            when (resource) {
                is Resource.Loading -> {
                    binding.progressBar.visibility = View.VISIBLE
                    binding.swipeRefreshLayout.isRefreshing = true
                }
                is Resource.Success -> {
                    binding.progressBar.visibility = View.GONE
                    binding.swipeRefreshLayout.isRefreshing = false
                    userAdapter.updateData(resource.data)
                }
                is Resource.Error -> {
                    binding.progressBar.visibility = View.GONE
                    binding.swipeRefreshLayout.isRefreshing = false
                    showError(resource.message)
                }
            }
        }
    }
    
    private fun setupRecyclerView() {
        userAdapter = UserAdapter(emptyList()) { user ->
            // Handle item click
            viewModel.fetchUserById(user.id)
        }
        
        binding.recyclerView.apply {
            adapter = userAdapter
            layoutManager = LinearLayoutManager(this@MainActivity)
        }
    }
    
    private fun showError(message: String) {
        Toast.makeText(this, message, Toast.LENGTH_LONG).show()
    }
}
```

## Best Practices

### 1. Error Handling
```kotlin
suspend fun safeApiCall(apiCall: suspend () -> Response<Any>): Resource<Any> {
    return try {
        val response = apiCall()
        if (response.isSuccessful) {
            Resource.Success(response.body()!!)
        } else {
            Resource.Error("HTTP ${response.code()}: ${response.message()}")
        }
    } catch (e: IOException) {
        Resource.Error("Network error: Check your internet connection")
    } catch (e: HttpException) {
        Resource.Error("Server error: ${e.message()}")
    } catch (e: Exception) {
        Resource.Error("Unknown error: ${e.message}")
    }
}
```

### 2. Authentication Interceptor
```kotlin
class AuthInterceptor(private val tokenProvider: () -> String?) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val originalRequest = chain.request()
        val token = tokenProvider()
        
        return if (token != null) {
            val authorizedRequest = originalRequest.newBuilder()
                .addHeader("Authorization", "Bearer $token")
                .build()
            chain.proceed(authorizedRequest)
        } else {
            chain.proceed(originalRequest)
        }
    }
}
```

Dengan implementasi ini, Anda memiliki setup Retrofit yang robust dan mudah untuk dipelihara!