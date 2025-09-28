---
title: "Tutorial Room Database: Implementasi Local Storage dengan Kotlin"
date: 2024-11-27
categories: ["android"]
tags: ["kotlin", "room", "database", "sqlite"]
featured_image: "images/posts/comp.jpg"
draft: false
---

Room adalah library persistence dari Android Architecture Components yang menyediakan abstraksi layer untuk SQLite. Pelajari implementasi lengkapnya dengan Kotlin.

## Setup Dependencies

Tambahkan dependencies di `build.gradle (Module: app)`:

```kotlin
dependencies {
    val room_version = "2.6.1"
    
    implementation "androidx.room:room-runtime:$room_version"
    implementation "androidx.room:room-ktx:$room_version"
    kapt "androidx.room:room-compiler:$room_version"
    
    // Coroutines
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3'
    
    // ViewModel dan LiveData
    implementation 'androidx.lifecycle:lifecycle-viewmodel-ktx:2.7.0'
    implementation 'androidx.lifecycle:lifecycle-livedata-ktx:2.7.0'
}
```

Enable kapt di `build.gradle (Module: app)`:

```kotlin
plugins {
    id 'kotlin-kapt'
}
```

## Entity (Data Model)

Buat entity untuk table database:

```kotlin
@Entity(tableName = "users")
data class User(
    @PrimaryKey(autoGenerate = true)
    val id: Int = 0,
    
    @ColumnInfo(name = "name")
    val name: String,
    
    @ColumnInfo(name = "email")
    val email: String,
    
    @ColumnInfo(name = "age")
    val age: Int,
    
    @ColumnInfo(name = "created_at")
    val createdAt: Long = System.currentTimeMillis()
)

@Entity(
    tableName = "notes",
    foreignKeys = [
        ForeignKey(
            entity = User::class,
            parentColumns = ["id"],
            childColumns = ["user_id"],
            onDelete = ForeignKey.CASCADE
        )
    ]
)
data class Note(
    @PrimaryKey(autoGenerate = true)
    val id: Int = 0,
    
    @ColumnInfo(name = "title")
    val title: String,
    
    @ColumnInfo(name = "content")
    val content: String,
    
    @ColumnInfo(name = "user_id")
    val userId: Int,
    
    @ColumnInfo(name = "created_at")
    val createdAt: Long = System.currentTimeMillis()
)
```

## Data Classes untuk Relations

```kotlin
data class UserWithNotes(
    @Embedded val user: User,
    @Relation(
        parentColumn = "id",
        entityColumn = "user_id"
    )
    val notes: List<Note>
)
```

## DAO (Data Access Object)

Definisikan operasi database:

```kotlin
@Dao
interface UserDao {
    
    @Query("SELECT * FROM users ORDER BY name ASC")
    fun getAllUsers(): Flow<List<User>>
    
    @Query("SELECT * FROM users WHERE id = :userId")
    suspend fun getUserById(userId: Int): User?
    
    @Query("SELECT * FROM users WHERE email = :email")
    suspend fun getUserByEmail(email: String): User?
    
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertUser(user: User): Long
    
    @Insert
    suspend fun insertUsers(users: List<User>)
    
    @Update
    suspend fun updateUser(user: User)
    
    @Delete
    suspend fun deleteUser(user: User)
    
    @Query("DELETE FROM users WHERE id = :userId")
    suspend fun deleteUserById(userId: Int)
    
    @Query("DELETE FROM users")
    suspend fun deleteAllUsers()
    
    // Search query
    @Query("SELECT * FROM users WHERE name LIKE '%' || :searchQuery || '%' OR email LIKE '%' || :searchQuery || '%'")
    fun searchUsers(searchQuery: String): Flow<List<User>>
}

@Dao
interface NoteDao {
    
    @Query("SELECT * FROM notes ORDER BY created_at DESC")
    fun getAllNotes(): Flow<List<Note>>
    
    @Query("SELECT * FROM notes WHERE user_id = :userId ORDER BY created_at DESC")
    fun getNotesByUserId(userId: Int): Flow<List<Note>>
    
    @Query("SELECT * FROM notes WHERE id = :noteId")
    suspend fun getNoteById(noteId: Int): Note?
    
    @Insert
    suspend fun insertNote(note: Note): Long
    
    @Update
    suspend fun updateNote(note: Note)
    
    @Delete
    suspend fun deleteNote(note: Note)
    
    @Query("DELETE FROM notes WHERE id = :noteId")
    suspend fun deleteNoteById(noteId: Int)
    
    // Get user with notes
    @Transaction
    @Query("SELECT * FROM users WHERE id = :userId")
    suspend fun getUserWithNotes(userId: Int): UserWithNotes?
    
    @Transaction
    @Query("SELECT * FROM users")
    fun getAllUsersWithNotes(): Flow<List<UserWithNotes>>
}
```

## Database Class

Implementasi Room Database:

```kotlin
@Database(
    entities = [User::class, Note::class],
    version = 1,
    exportSchema = false
)
@TypeConverters(Converters::class)
abstract class AppDatabase : RoomDatabase() {
    
    abstract fun userDao(): UserDao
    abstract fun noteDao(): NoteDao
    
    companion object {
        @Volatile
        private var INSTANCE: AppDatabase? = null
        
        fun getDatabase(context: Context): AppDatabase {
            return INSTANCE ?: synchronized(this) {
                val instance = Room.databaseBuilder(
                    context.applicationContext,
                    AppDatabase::class.java,
                    "app_database"
                )
                    .fallbackToDestructiveMigration()
                    .build()
                INSTANCE = instance
                instance
            }
        }
    }
}

// Type Converters untuk data types yang complex
class Converters {
    @TypeConverter
    fun fromTimestamp(value: Long?): Date? {
        return value?.let { Date(it) }
    }
    
    @TypeConverter
    fun dateToTimestamp(date: Date?): Long? {
        return date?.time
    }
}
```

## Repository Pattern

Implementasi repository untuk data management:

```kotlin
class UserRepository(private val userDao: UserDao, private val noteDao: NoteDao) {
    
    fun getAllUsers(): Flow<List<User>> = userDao.getAllUsers()
    
    suspend fun getUserById(userId: Int): User? = userDao.getUserById(userId)
    
    suspend fun insertUser(user: User): Long = userDao.insertUser(user)
    
    suspend fun updateUser(user: User) = userDao.updateUser(user)
    
    suspend fun deleteUser(user: User) = userDao.deleteUser(user)
    
    fun searchUsers(query: String): Flow<List<User>> = userDao.searchUsers(query)
    
    // Note operations
    fun getAllNotes(): Flow<List<Note>> = noteDao.getAllNotes()
    
    fun getNotesByUserId(userId: Int): Flow<List<Note>> = noteDao.getNotesByUserId(userId)
    
    suspend fun insertNote(note: Note): Long = noteDao.insertNote(note)
    
    suspend fun updateNote(note: Note) = noteDao.updateNote(note)
    
    suspend fun deleteNote(note: Note) = noteDao.deleteNote(note)
    
    // Relations
    suspend fun getUserWithNotes(userId: Int): UserWithNotes? = noteDao.getUserWithNotes(userId)
    
    fun getAllUsersWithNotes(): Flow<List<UserWithNotes>> = noteDao.getAllUsersWithNotes()
}
```

## ViewModel Implementation

```kotlin
class UserViewModel(application: Application) : AndroidViewModel(application) {
    
    private val repository: UserRepository
    
    init {
        val database = AppDatabase.getDatabase(application)
        repository = UserRepository(database.userDao(), database.noteDao())
    }
    
    val allUsers: LiveData<List<User>> = repository.getAllUsers().asLiveData()
    val allUsersWithNotes: LiveData<List<UserWithNotes>> = repository.getAllUsersWithNotes().asLiveData()
    
    private val _searchQuery = MutableLiveData<String>()
    val searchResults: LiveData<List<User>> = _searchQuery.switchMap { query ->
        if (query.isBlank()) {
            repository.getAllUsers().asLiveData()
        } else {
            repository.searchUsers(query).asLiveData()
        }
    }
    
    fun insertUser(user: User) = viewModelScope.launch {
        repository.insertUser(user)
    }
    
    fun updateUser(user: User) = viewModelScope.launch {
        repository.updateUser(user)
    }
    
    fun deleteUser(user: User) = viewModelScope.launch {
        repository.deleteUser(user)
    }
    
    fun searchUsers(query: String) {
        _searchQuery.value = query
    }
    
    fun insertNote(note: Note) = viewModelScope.launch {
        repository.insertNote(note)
    }
    
    fun getUserWithNotes(userId: Int): LiveData<UserWithNotes?> = liveData {
        emit(repository.getUserWithNotes(userId))
    }
}

class UserViewModelFactory(private val application: Application) : ViewModelProvider.Factory {
    override fun <T : ViewModel> create(modelClass: Class<T>): T {
        if (modelClass.isAssignableFrom(UserViewModel::class.java)) {
            @Suppress("UNCHECKED_CAST")
            return UserViewModel(application) as T
        }
        throw IllegalArgumentException("Unknown ViewModel class")
    }
}
```

## Activity Implementation

Menggunakan Room database di Activity:

```kotlin
class MainActivity : AppCompatActivity() {
    
    private lateinit var binding: ActivityMainBinding
    private lateinit var userAdapter: UserAdapter
    private lateinit var viewModel: UserViewModel
    
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)
        
        val factory = UserViewModelFactory(application)
        viewModel = ViewModelProvider(this, factory)[UserViewModel::class.java]
        
        setupRecyclerView()
        observeViewModel()
        setupSearchView()
        
        binding.fabAddUser.setOnClickListener {
            showAddUserDialog()
        }
    }
    
    private fun setupRecyclerView() {
        userAdapter = UserAdapter(
            onItemClick = { user -> showUserDetail(user) },
            onDeleteClick = { user -> deleteUser(user) }
        )
        
        binding.recyclerView.apply {
            adapter = userAdapter
            layoutManager = LinearLayoutManager(this@MainActivity)
        }
    }
    
    private fun observeViewModel() {
        viewModel.searchResults.observe(this) { users ->
            userAdapter.submitList(users)
        }
    }
    
    private fun setupSearchView() {
        binding.searchView.setOnQueryTextListener(object : SearchView.OnQueryTextListener {
            override fun onQueryTextSubmit(query: String?): Boolean {
                return false
            }
            
            override fun onQueryTextChange(newText: String?): Boolean {
                viewModel.searchUsers(newText ?: "")
                return true
            }
        })
    }
    
    private fun showAddUserDialog() {
        val dialogBinding = DialogAddUserBinding.inflate(layoutInflater)
        
        AlertDialog.Builder(this)
            .setTitle("Add New User")
            .setView(dialogBinding.root)
            .setPositiveButton("Add") { _, _ ->
                val name = dialogBinding.etName.text.toString()
                val email = dialogBinding.etEmail.text.toString()
                val age = dialogBinding.etAge.text.toString().toIntOrNull() ?: 0
                
                if (name.isNotBlank() && email.isNotBlank()) {
                    val user = User(name = name, email = email, age = age)
                    viewModel.insertUser(user)
                }
            }
            .setNegativeButton("Cancel", null)
            .show()
    }
    
    private fun showUserDetail(user: User) {
        viewModel.getUserWithNotes(user.id).observe(this) { userWithNotes ->
            userWithNotes?.let { 
                // Navigate to detail activity or show detail dialog
                val intent = Intent(this, UserDetailActivity::class.java).apply {
                    putExtra("USER_ID", user.id)
                }
                startActivity(intent)
            }
        }
    }
    
    private fun deleteUser(user: User) {
        AlertDialog.Builder(this)
            .setTitle("Delete User")
            .setMessage("Are you sure you want to delete ${user.name}?")
            .setPositiveButton("Delete") { _, _ ->
                viewModel.deleteUser(user)
            }
            .setNegativeButton("Cancel", null)
            .show()
    }
}
```

## Database Migration

Untuk update database schema:

```kotlin
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(database: SupportSQLiteDatabase) {
        database.execSQL("ALTER TABLE users ADD COLUMN phone TEXT DEFAULT ''")
    }
}

// Tambahkan migration ke database builder
Room.databaseBuilder(context, AppDatabase::class.java, "app_database")
    .addMigrations(MIGRATION_1_2)
    .build()
```

Dengan implementasi Room Database ini, Anda memiliki local storage yang powerful dan type-safe untuk aplikasi Android Anda!