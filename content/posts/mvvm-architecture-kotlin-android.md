---
title: "Implementasi MVVM Architecture Pattern dengan Kotlin di Android"
date: 2024-11-26
categories: ["android"]
tags: ["kotlin", "mvvm", "architecture", "clean-code"]
featured_image: "images/posts/comp.jpg"
draft: false
---

MVVM (Model-View-ViewModel) adalah pattern arsitektur yang direkomendasikan Google untuk pengembangan Android. Pelajari implementasi lengkap dengan Kotlin.

## Konsep MVVM Architecture

MVVM terdiri dari tiga komponen utama:
- **Model**: Data layer (Repository, Database, Network)
- **View**: UI layer (Activity, Fragment, XML layouts)
- **ViewModel**: Business logic yang menghubungkan Model dan View

## Setup Dependencies

Tambahkan dependencies di `build.gradle (Module: app)`:

```kotlin
dependencies {
    // ViewModel dan LiveData
    implementation 'androidx.lifecycle:lifecycle-viewmodel-ktx:2.7.0'
    implementation 'androidx.lifecycle:lifecycle-livedata-ktx:2.7.0'
    implementation 'androidx.activity:activity-ktx:1.8.2'
    implementation 'androidx.fragment:fragment-ktx:1.6.2'
    
    // Data Binding
    implementation 'androidx.databinding:databinding-runtime:8.2.0'
    
    // Coroutines
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3'
}
```

Enable data binding di `build.gradle (Module: app)`:

```kotlin
android {
    buildFeatures {
        dataBinding = true
        viewBinding = true
    }
}
```

## Model Layer

### Data Class Models
```kotlin
data class User(
    val id: Int,
    val name: String,
    val email: String,
    val avatar: String,
    val isOnline: Boolean = false
)

data class Post(
    val id: Int,
    val userId: Int,
    val title: String,
    val content: String,
    val createdAt: Long = System.currentTimeMillis()
)

sealed class NetworkResult<T> {
    data class Success<T>(val data: T) : NetworkResult<T>()
    data class Error<T>(val message: String) : NetworkResult<T>()
    data class Loading<T>(val data: T? = null) : NetworkResult<T>()
}
```

### Repository Pattern
```kotlin
interface UserRepository {
    suspend fun getUsers(): NetworkResult<List<User>>
    suspend fun getUserById(id: Int): NetworkResult<User>
    suspend fun createUser(user: User): NetworkResult<User>
    suspend fun updateUser(user: User): NetworkResult<User>
    suspend fun deleteUser(id: Int): NetworkResult<Unit>
}

class UserRepositoryImpl(
    private val apiService: ApiService,
    private val localDataSource: UserDao
) : UserRepository {
    
    override suspend fun getUsers(): NetworkResult<List<User>> {
        return try {
            val response = apiService.getUsers()
            if (response.isSuccessful) {
                response.body()?.let { users ->
                    // Cache to local database
                    localDataSource.insertUsers(users)
                    NetworkResult.Success(users)
                } ?: NetworkResult.Error("Empty response")
            } else {
                // Fallback to local data
                val localUsers = localDataSource.getAllUsers()
                if (localUsers.isNotEmpty()) {
                    NetworkResult.Success(localUsers)
                } else {
                    NetworkResult.Error("No data available")
                }
            }
        } catch (e: Exception) {
            // Return cached data on network error
            val cachedUsers = localDataSource.getAllUsers()
            if (cachedUsers.isNotEmpty()) {
                NetworkResult.Success(cachedUsers)
            } else {
                NetworkResult.Error("Network error: ${e.message}")
            }
        }
    }
    
    override suspend fun getUserById(id: Int): NetworkResult<User> {
        return try {
            val response = apiService.getUserById(id)
            if (response.isSuccessful) {
                response.body()?.let { user ->
                    localDataSource.insertUser(user)
                    NetworkResult.Success(user)
                } ?: NetworkResult.Error("User not found")
            } else {
                NetworkResult.Error("Error: ${response.message()}")
            }
        } catch (e: Exception) {
            NetworkResult.Error("Network error: ${e.message}")
        }
    }
    
    override suspend fun createUser(user: User): NetworkResult<User> {
        return try {
            val response = apiService.createUser(user)
            if (response.isSuccessful) {
                response.body()?.let { createdUser ->
                    localDataSource.insertUser(createdUser)
                    NetworkResult.Success(createdUser)
                } ?: NetworkResult.Error("Failed to create user")
            } else {
                NetworkResult.Error("Error: ${response.message()}")
            }
        } catch (e: Exception) {
            NetworkResult.Error("Network error: ${e.message}")
        }
    }
    
    override suspend fun updateUser(user: User): NetworkResult<User> {
        return try {
            val response = apiService.updateUser(user.id, user)
            if (response.isSuccessful) {
                response.body()?.let { updatedUser ->
                    localDataSource.updateUser(updatedUser)
                    NetworkResult.Success(updatedUser)
                } ?: NetworkResult.Error("Failed to update user")
            } else {
                NetworkResult.Error("Error: ${response.message()}")
            }
        } catch (e: Exception) {
            NetworkResult.Error("Network error: ${e.message}")
        }
    }
    
    override suspend fun deleteUser(id: Int): NetworkResult<Unit> {
        return try {
            val response = apiService.deleteUser(id)
            if (response.isSuccessful) {
                localDataSource.deleteUserById(id)
                NetworkResult.Success(Unit)
            } else {
                NetworkResult.Error("Failed to delete user")
            }
        } catch (e: Exception) {
            NetworkResult.Error("Network error: ${e.message}")
        }
    }
}
```

## ViewModel Layer

### Base ViewModel
```kotlin
abstract class BaseViewModel : ViewModel() {
    
    private val _loading = MutableLiveData<Boolean>()
    val loading: LiveData<Boolean> = _loading
    
    private val _error = MutableLiveData<String>()
    val error: LiveData<String> = _error
    
    protected fun setLoading(isLoading: Boolean) {
        _loading.value = isLoading
    }
    
    protected fun setError(message: String) {
        _error.value = message
    }
    
    protected fun clearError() {
        _error.value = ""
    }
}
```

### User ViewModel
```kotlin
class UserViewModel(
    private val userRepository: UserRepository
) : BaseViewModel() {
    
    private val _users = MutableLiveData<List<User>>()
    val users: LiveData<List<User>> = _users
    
    private val _selectedUser = MutableLiveData<User>()
    val selectedUser: LiveData<User> = _selectedUser
    
    private val _isRefreshing = MutableLiveData<Boolean>()
    val isRefreshing: LiveData<Boolean> = _isRefreshing
    
    // Search functionality
    private val _searchQuery = MutableLiveData<String>()
    val searchQuery: LiveData<String> = _searchQuery
    
    val filteredUsers: LiveData<List<User>> = _searchQuery.switchMap { query ->
        liveData {
            val allUsers = _users.value ?: emptyList()
            if (query.isNullOrBlank()) {
                emit(allUsers)
            } else {
                emit(allUsers.filter { user ->
                    user.name.contains(query, ignoreCase = true) ||
                    user.email.contains(query, ignoreCase = true)
                })
            }
        }
    }
    
    fun loadUsers() {
        viewModelScope.launch {
            setLoading(true)
            when (val result = userRepository.getUsers()) {
                is NetworkResult.Success -> {
                    _users.value = result.data
                    clearError()
                }
                is NetworkResult.Error -> {
                    setError(result.message)
                }
                is NetworkResult.Loading -> {
                    // Handle loading state if needed
                }
            }
            setLoading(false)
        }
    }
    
    fun refreshUsers() {
        viewModelScope.launch {
            _isRefreshing.value = true
            when (val result = userRepository.getUsers()) {
                is NetworkResult.Success -> {
                    _users.value = result.data
                    clearError()
                }
                is NetworkResult.Error -> {
                    setError(result.message)
                }
                is NetworkResult.Loading -> {
                    // Handle loading state if needed
                }
            }
            _isRefreshing.value = false
        }
    }
    
    fun loadUserById(userId: Int) {
        viewModelScope.launch {
            setLoading(true)
            when (val result = userRepository.getUserById(userId)) {
                is NetworkResult.Success -> {
                    _selectedUser.value = result.data
                    clearError()
                }
                is NetworkResult.Error -> {
                    setError(result.message)
                }
                is NetworkResult.Loading -> {
                    // Handle loading state if needed
                }
            }
            setLoading(false)
        }
    }
    
    fun createUser(name: String, email: String) {
        if (name.isBlank() || email.isBlank()) {
            setError("Name and email are required")
            return
        }
        
        viewModelScope.launch {
            setLoading(true)
            val newUser = User(
                id = 0, // Will be assigned by server
                name = name,
                email = email,
                avatar = ""
            )
            
            when (val result = userRepository.createUser(newUser)) {
                is NetworkResult.Success -> {
                    // Refresh the list
                    loadUsers()
                    clearError()
                }
                is NetworkResult.Error -> {
                    setError(result.message)
                }
                is NetworkResult.Loading -> {
                    // Handle loading state if needed
                }
            }
            setLoading(false)
        }
    }
    
    fun updateUser(user: User) {
        viewModelScope.launch {
            setLoading(true)
            when (val result = userRepository.updateUser(user)) {
                is NetworkResult.Success -> {
                    _selectedUser.value = result.data
                    loadUsers() // Refresh list
                    clearError()
                }
                is NetworkResult.Error -> {
                    setError(result.message)
                }
                is NetworkResult.Loading -> {
                    // Handle loading state if needed
                }
            }
            setLoading(false)
        }
    }
    
    fun deleteUser(userId: Int) {
        viewModelScope.launch {
            setLoading(true)
            when (val result = userRepository.deleteUser(userId)) {
                is NetworkResult.Success -> {
                    loadUsers() // Refresh list
                    clearError()
                }
                is NetworkResult.Error -> {
                    setError(result.message)
                }
                is NetworkResult.Loading -> {
                    // Handle loading state if needed
                }
            }
            setLoading(false)
        }
    }
    
    fun searchUsers(query: String) {
        _searchQuery.value = query
    }
    
    fun selectUser(user: User) {
        _selectedUser.value = user
    }
}

class UserViewModelFactory(
    private val userRepository: UserRepository
) : ViewModelProvider.Factory {
    override fun <T : ViewModel> create(modelClass: Class<T>): T {
        if (modelClass.isAssignableFrom(UserViewModel::class.java)) {
            @Suppress("UNCHECKED_CAST")
            return UserViewModel(userRepository) as T
        }
        throw IllegalArgumentException("Unknown ViewModel class")
    }
}
```

## View Layer

### Activity with Data Binding
```kotlin
class MainActivity : AppCompatActivity() {
    
    private lateinit var binding: ActivityMainBinding
    private lateinit var viewModel: UserViewModel
    private lateinit var userAdapter: UserAdapter
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = DataBindingUtil.setContentView(this, R.layout.activity_main)
        
        setupViewModel()
        setupDataBinding()
        setupRecyclerView()
        observeViewModel()
        
        // Initial load
        viewModel.loadUsers()
    }
    
    private fun setupViewModel() {
        val apiService = RetrofitClient.instance
        val database = AppDatabase.getDatabase(this)
        val repository = UserRepositoryImpl(apiService, database.userDao())
        val factory = UserViewModelFactory(repository)
        
        viewModel = ViewModelProvider(this, factory)[UserViewModel::class.java]
    }
    
    private fun setupDataBinding() {
        binding.apply {
            lifecycleOwner = this@MainActivity
            viewModel = this@MainActivity.viewModel
            
            // Setup search functionality
            searchView.setOnQueryTextListener(object : SearchView.OnQueryTextListener {
                override fun onQueryTextSubmit(query: String?): Boolean = false
                
                override fun onQueryTextChange(newText: String?): Boolean {
                    viewModel?.searchUsers(newText ?: "")
                    return true
                }
            })
            
            // Setup swipe refresh
            swipeRefreshLayout.setOnRefreshListener {
                viewModel?.refreshUsers()
            }
            
            // Setup FAB
            fabAddUser.setOnClickListener {
                showAddUserDialog()
            }
        }
    }
    
    private fun setupRecyclerView() {
        userAdapter = UserAdapter(
            onItemClick = { user -> 
                viewModel.selectUser(user)
                navigateToUserDetail(user)
            },
            onDeleteClick = { user ->
                showDeleteConfirmation(user)
            }
        )
        
        binding.recyclerView.apply {
            adapter = userAdapter
            layoutManager = LinearLayoutManager(this@MainActivity)
        }
    }
    
    private fun observeViewModel() {
        viewModel.filteredUsers.observe(this) { users ->
            userAdapter.submitList(users)
        }
        
        viewModel.loading.observe(this) { isLoading ->
            binding.progressBar.visibility = if (isLoading) View.VISIBLE else View.GONE
        }
        
        viewModel.isRefreshing.observe(this) { isRefreshing ->
            binding.swipeRefreshLayout.isRefreshing = isRefreshing
        }
        
        viewModel.error.observe(this) { errorMessage ->
            if (errorMessage.isNotEmpty()) {
                showError(errorMessage)
            }
        }
    }
    
    private fun showAddUserDialog() {
        val dialogBinding = DialogAddUserBinding.inflate(layoutInflater)
        
        val dialog = AlertDialog.Builder(this)
            .setTitle("Add New User")
            .setView(dialogBinding.root)
            .setPositiveButton("Add") { _, _ ->
                val name = dialogBinding.etName.text.toString()
                val email = dialogBinding.etEmail.text.toString()
                viewModel.createUser(name, email)
            }
            .setNegativeButton("Cancel", null)
            .create()
        
        dialog.show()
    }
    
    private fun showDeleteConfirmation(user: User) {
        AlertDialog.Builder(this)
            .setTitle("Delete User")
            .setMessage("Are you sure you want to delete ${user.name}?")
            .setPositiveButton("Delete") { _, _ ->
                viewModel.deleteUser(user.id)
            }
            .setNegativeButton("Cancel", null)
            .show()
    }
    
    private fun navigateToUserDetail(user: User) {
        val intent = Intent(this, UserDetailActivity::class.java).apply {
            putExtra("USER_ID", user.id)
        }
        startActivity(intent)
    }
    
    private fun showError(message: String) {
        Snackbar.make(binding.root, message, Snackbar.LENGTH_LONG).show()
    }
}
```

### Layout dengan Data Binding
```xml
<layout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools">

    <data>
        <variable
            name="viewModel"
            type="com.example.mvvm.UserViewModel" />
    </data>

    <androidx.coordinatorlayout.widget.CoordinatorLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <androidx.swiperefreshlayout.widget.SwipeRefreshLayout
            android:id="@+id/swipeRefreshLayout"
            android:layout_width="match_parent"
            android:layout_height="match_parent">

            <LinearLayout
                android:layout_width="match_parent"
                android:layout_height="match_parent"
                android:orientation="vertical">

                <androidx.appcompat.widget.SearchView
                    android:id="@+id/searchView"
                    android:layout_width="match_parent"
                    android:layout_height="wrap_content"
                    android:layout_margin="16dp"
                    app:queryHint="Search users..." />

                <androidx.recyclerview.widget.RecyclerView
                    android:id="@+id/recyclerView"
                    android:layout_width="match_parent"
                    android:layout_height="0dp"
                    android:layout_weight="1"
                    tools:listitem="@layout/item_user" />

            </LinearLayout>

        </androidx.swiperefreshlayout.widget.SwipeRefreshLayout>

        <ProgressBar
            android:id="@+id/progressBar"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_gravity="center"
            android:visibility="gone" />

        <com.google.android.material.floatingactionbutton.FloatingActionButton
            android:id="@+id/fabAddUser"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:layout_gravity="bottom|end"
            android:layout_margin="16dp"
            android:src="@drawable/ic_add" />

    </androidx.coordinatorlayout.widget.CoordinatorLayout>

</layout>
```

## Testing ViewModel

```kotlin
@ExperimentalCoroutinesApi
class UserViewModelTest {
    
    @get:Rule
    val instantExecutorRule = InstantTaskExecutorRule()
    
    @get:Rule
    val mainDispatcherRule = MainDispatcherRule()
    
    private lateinit var viewModel: UserViewModel
    private lateinit var mockRepository: UserRepository
    
    @Before
    fun setup() {
        mockRepository = mockk()
        viewModel = UserViewModel(mockRepository)
    }
    
    @Test
    fun `loadUsers success updates users LiveData`() = runTest {
        // Given
        val users = listOf(
            User(1, "John", "john@example.com", ""),
            User(2, "Jane", "jane@example.com", "")
        )
        coEvery { mockRepository.getUsers() } returns NetworkResult.Success(users)
        
        // When
        viewModel.loadUsers()
        
        // Then
        assertEquals(users, viewModel.users.getOrAwaitValue())
        assertEquals(false, viewModel.loading.getOrAwaitValue())
    }
    
    @Test
    fun `loadUsers error updates error LiveData`() = runTest {
        // Given
        val errorMessage = "Network error"
        coEvery { mockRepository.getUsers() } returns NetworkResult.Error(errorMessage)
        
        // When
        viewModel.loadUsers()
        
        // Then
        assertEquals(errorMessage, viewModel.error.getOrAwaitValue())
        assertEquals(false, viewModel.loading.getOrAwaitValue())
    }
}
```

Dengan implementasi MVVM yang komprehensif ini, Anda memiliki arsitektur yang terstruktur, testable, dan mudah dipelihara untuk aplikasi Android!