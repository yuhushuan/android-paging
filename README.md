https://github.com/googlecodelabs/android-paging/issues/36

提示AndroidStudio使用4.0以上，修改工程下`build.gradle`文件

```groovy
buildscript {
    dependencies {
        classpath 'com.android.tools.build:gradle:3.5.3'
    }
}
```

**获取 SearchRepositoriesViewModel 实例：**

- 通过自定义 ViewModelProvider.Factory 获取 SearchRepositoriesViewModel 实例

  ```kotlin
  // get the view model
  viewModel = ViewModelProviders.of(this, Injection.provideViewModelFactory(this))
          .get(SearchRepositoriesViewModel::class.java)
  ```
  
- Injection.provideViewModelFactory(this)

  ```kotlin
      /**
       * Creates an instance of [GithubLocalCache] based on the database DAO.
       */
      private fun provideCache(context: Context): GithubLocalCache {
          val database = RepoDatabase.getInstance(context)
          return GithubLocalCache(database.reposDao(), Executors.newSingleThreadExecutor())
      }
  
      /**
       * Creates an instance of [GithubRepository] based on the [GithubService] and a
       * [GithubLocalCache]
       */
      private fun provideGithubRepository(context: Context): GithubRepository {
          return GithubRepository(GithubService.create(), provideCache(context))
      }
  
      /**
       * Provides the [ViewModelProvider.Factory] that is then used to get a reference to
       * [ViewModel] objects.
       */
      fun provideViewModelFactory(context: Context): ViewModelProvider.Factory {
          return ViewModelFactory(provideGithubRepository(context))
      }
  ```

- ViewModelFactory

  ```kotlin
      override fun <T : ViewModel> create(modelClass: Class<T>): T {
          if (modelClass.isAssignableFrom(SearchRepositoriesViewModel::class.java)) {
              @Suppress("UNCHECKED_CAST")
              return SearchRepositoriesViewModel(repository) as T
          }
          throw IllegalArgumentException("Unknown ViewModel class")
      }
  ```

**获取数据**

1. 通过查询关键字搜索 `queryLiveData.postValue(queryString)`

2. 根据关键字获取数据 `repository.search(it)`

   ```kotlin
   private val repoResult: LiveData<RepoSearchResult> = Transformations.map(queryLiveData) {
       repository.search(it)
   }
   ```

3. 第一页从数据库获取数据，同时请求网络数据并保持到数据库

   ```kotlin
   /**
    * Search repositories whose names match the query.
    */
   fun search(query: String): RepoSearchResult {
       Log.d("GithubRepository", "New query: $query")
       lastRequestedPage = 1
       requestAndSaveData(query)
   
       // Get data from the local cache
       val data = cache.reposByName(query)
   
       return RepoSearchResult(data, networkErrors)
   }
   ```

4. 观察返回的数据

   ```kotlin
   viewModel.repos.observe(this, Observer<List<Repo>> {
       Log.d("Activity", "list: ${it?.size}")
       showEmptyList(it?.size == 0)
       adapter.submitList(it)
   })
   ```

5. 请求更多数据

   ```kotlin
   fun requestMore(query: String) {
       requestAndSaveData(query)
   }
   ```
