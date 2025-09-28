---
title: "Tutorial Lengkap: Membuat RecyclerView dengan Kotlin di Android"
date: 2024-11-30
categories: ["android"]
tags: ["kotlin", "android", "recyclerview", "tutorial"]
featured_image: "images/posts/comp.jpg"
draft: false
---

RecyclerView adalah komponen penting dalam pengembangan Android untuk menampilkan daftar data yang efisien. Pelajari cara implementasinya dengan Kotlin.

## Setup Dependencies

Tambahkan dependency di `build.gradle (Module: app)`:

```kotlin
dependencies {
    implementation 'androidx.recyclerview:recyclerview:1.3.2'
    implementation 'androidx.cardview:cardview:1.0.0'
}
```

## Model Data Class

Buat data class untuk item yang akan ditampilkan:

```kotlin
data class User(
    val id: Int,
    val name: String,
    val email: String,
    val profileImage: String
)
```

## Layout Item RecyclerView

Buat file `item_user.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.cardview.widget.CardView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_margin="8dp"
    app:cardCornerRadius="8dp"
    app:cardElevation="4dp">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:padding="16dp">

        <ImageView
            android:id="@+id/ivProfile"
            android:layout_width="60dp"
            android:layout_height="60dp"
            android:layout_marginEnd="16dp"
            android:scaleType="centerCrop" />

        <LinearLayout
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:orientation="vertical">

            <TextView
                android:id="@+id/tvName"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:textSize="18sp"
                android:textStyle="bold" />

            <TextView
                android:id="@+id/tvEmail"
                android:layout_width="match_parent"
                android:layout_height="wrap_content"
                android:textColor="@android:color/darker_gray"
                android:textSize="14sp" />

        </LinearLayout>

    </LinearLayout>

</androidx.cardview.widget.CardView>
```

## Adapter RecyclerView

Implementasi adapter dengan Kotlin:

```kotlin
class UserAdapter(
    private var userList: List<User>,
    private val onItemClick: (User) -> Unit
) : RecyclerView.Adapter<UserAdapter.UserViewHolder>() {

    inner class UserViewHolder(private val binding: ItemUserBinding) : 
        RecyclerView.ViewHolder(binding.root) {
        
        fun bind(user: User) {
            binding.apply {
                tvName.text = user.name
                tvEmail.text = user.email
                
                // Load image menggunakan Glide
                Glide.with(itemView.context)
                    .load(user.profileImage)
                    .placeholder(R.drawable.ic_user_placeholder)
                    .into(ivProfile)
                
                root.setOnClickListener { onItemClick(user) }
            }
        }
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): UserViewHolder {
        val binding = ItemUserBinding.inflate(
            LayoutInflater.from(parent.context), 
            parent, 
            false
        )
        return UserViewHolder(binding)
    }

    override fun onBindViewHolder(holder: UserViewHolder, position: Int) {
        holder.bind(userList[position])
    }

    override fun getItemCount(): Int = userList.size

    fun updateData(newList: List<User>) {
        userList = newList
        notifyDataSetChanged()
    }
}
```

## Setup di Activity

Implementasi RecyclerView di MainActivity:

```kotlin
class MainActivity : AppCompatActivity() {
    
    private lateinit var binding: ActivityMainBinding
    private lateinit var userAdapter: UserAdapter
    private val userList = mutableListOf<User>()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(layoutInflater)
        setContentView(binding.root)
        
        setupRecyclerView()
        loadUserData()
    }

    private fun setupRecyclerView() {
        userAdapter = UserAdapter(userList) { user ->
            // Handle item click
            showUserDetail(user)
        }
        
        binding.recyclerView.apply {
            adapter = userAdapter
            layoutManager = LinearLayoutManager(this@MainActivity)
            
            // Add item decoration untuk spacing
            addItemDecoration(
                DividerItemDecoration(
                    this@MainActivity,
                    DividerItemDecoration.VERTICAL
                )
            )
        }
    }

    private fun loadUserData() {
        // Simulasi loading data
        val users = listOf(
            User(1, "John Doe", "john@example.com", "https://example.com/avatar1.jpg"),
            User(2, "Jane Smith", "jane@example.com", "https://example.com/avatar2.jpg"),
            User(3, "Bob Johnson", "bob@example.com", "https://example.com/avatar3.jpg")
        )
        
        userList.clear()
        userList.addAll(users)
        userAdapter.notifyDataSetChanged()
    }

    private fun showUserDetail(user: User) {
        Toast.makeText(this, "Selected: ${user.name}", Toast.LENGTH_SHORT).show()
        // Navigate to detail activity
        val intent = Intent(this, UserDetailActivity::class.java).apply {
            putExtra("USER_ID", user.id)
        }
        startActivity(intent)
    }
}
```

## Optimasi dengan DiffUtil

Untuk performa yang lebih baik, gunakan DiffUtil:

```kotlin
class UserDiffCallback(
    private val oldList: List<User>,
    private val newList: List<User>
) : DiffUtil.Callback() {

    override fun getOldListSize(): Int = oldList.size

    override fun getNewListSize(): Int = newList.size

    override fun areItemsTheSame(oldItemPosition: Int, newItemPosition: Int): Boolean {
        return oldList[oldItemPosition].id == newList[newItemPosition].id
    }

    override fun areContentsTheSame(oldItemPosition: Int, newItemPosition: Int): Boolean {
        return oldList[oldItemPosition] == newList[newItemPosition]
    }
}

// Update method di adapter
fun updateDataWithDiffUtil(newList: List<User>) {
    val diffCallback = UserDiffCallback(userList, newList)
    val diffResult = DiffUtil.calculateDiff(diffCallback)
    
    userList = newList.toMutableList()
    diffResult.dispatchUpdatesTo(this)
}
```

Dengan implementasi ini, Anda memiliki RecyclerView yang efisien dan responsif untuk menampilkan daftar data di aplikasi Android Anda!