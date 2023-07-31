# eloquent-performance-patterns
this is the personalized readme section from this user


## PS

Always be careful while using EAGER LOADING, check out the section 3.

## Lesson 1

you can add index to the column which you want to use by orderBy.

even memory usage is looks normal should check out the models metric.

## Minimize Memory Usage - 2

you should specify if you need/can the columns, while fetching the data to decrease the memory usage.

e.g:
            ->select('id', 'title', 'slug', 'published_at', 'author_id')
            ->with('author:id,name')
            ->latest('published_at')

## Getting One Record From A Has Many Relationship - 3

tables: users-logins

You are displaying the users from the users table, but what if you want the latest login date from the logins table which there is tons of login record belongs to the users? 

solution one: run a query for each user. e.g: {{ $user->logins()->latest()->first()->created_at->diffForHumans() }}
however, it'll cause to N+1 issue. If our page displays 10 users, it will send 12 queries.

the wrong way: eager loading on your controller. e.g: User::query()->with('logins'), on your blade: {{ $user->logins->sortByDesc('created_at')->first()->created_at->diffForHumans() }}. It may decrease your queries, but it'll fetch all the login records belongs to the users. This is critical.

the right way: using subqueries,

on your controller. e.g: User::query()->addSelect(['last_login_at' => Login::select('created_at')->whereColumn('user_id', 'users.id')->latest()->take(1)]),

on your blade e.g: {{ $user->last_login_at }}

it is being computed on the fly in our subquery. AS: user columns...column1...column2...last_login_at

one step further, scoping:
on your user model, e.g:
    public function scopeWithLastLoginAt($query)
    {
        $query->addSelect(['last_login_at' => Login::select('created_at')
                ->whereColumn('user_id', 'users.id')
                ->latest()
                ->take(1)
            ])
            ->withCasts(['last_login_at' => 'datetime']);
    }
on your users controller, e.g:
          User::query()
            ->select('name', 'email', 'id')
            ->withLastLoginAt()
            ->orderBy('name')
            ->paginate();
            
additional: subqueries can only return single column, so that is why we used take(1) attribute.


## Dynamic Relationships Using Subqueries - 4

We can take single column via section 3, but what if we want multiple column?

edit the user model as.

user model e.g:

public function lastLogin()
{
  return $this->belongsTo(Login::class);
}

public function scopeWithLastLogin()
{
  $query->addSelect(['last_login_id' => Login::select('id')
      ->whereColumn('user_id', 'users.id')
      ->latest()
      -> take(1)
      ])->with('lastLogin');
}

user controller e.g:
          User::query()
            ->select('name', 'email', 'id')
            ->withLastLogin()
            ->orderBy('name')
            ->paginate();
user blade e.g:
{{ $user->lastLogin->created_at->diffForHumans() }}
{{ $user->lastLogin->ip_address }} 

additional: you can not lazy load dynamic relationships

Eager VS Lazy Load
<br>
Laravel's lazy loading is a feature that allows you to defer the loading of related models or relationships until you actually access them. In other words, when retrieving a model from the database, related models are not loaded immediately, but they are fetched from the database only when you request them.

By default, when you eager load relationships using methods like `with()`, Laravel retrieves all the related records and maps them to the appropriate relationships. While this approach is efficient for many use cases, it can lead to unnecessary database queries and memory consumption when you don't actually need the related data.

Lazy loading helps optimize the performance and memory usage by fetching related data on-demand, only when it's needed. This can be especially useful when dealing with large datasets or complex relationships. Instead of pulling all the related records at once, lazy loading queries the database as you access the relationships.

Here's an example to illustrate the difference between eager loading and lazy loading:

1. Eager Loading (Default):
```php
// Eager loading - retrieves all the posts and their comments in one query
$posts = Post::with('comments')->get();

foreach ($posts as $post) {
    // Comments are already loaded and available
    foreach ($post->comments as $comment) {
        // Do something with the comments
    }
}
```

2. Lazy Loading:
```php
// Lazy loading - retrieves only the posts first, then fetches comments for each post as needed
$posts = Post::all();

foreach ($posts as $post) {
    // Lazy loading of comments, only fetched when accessed
    foreach ($post->comments as $comment) {
        // Do something with the comments
    }
}
```

In the second example, comments are loaded from the database only when `$post->comments` is accessed in the loop.

To enable lazy loading in Laravel, you don't need to do anything special. It's the default behavior for relationships. However, when using eager loading, you can explicitly disable it for certain relationships by specifying them as "lazy" in the `with()` method.

```php
// Eager loading with 'lazy' relationship
$posts = Post::with('comments:lazy')->get();
```

Overall, Laravel's lazy loading helps improve the performance of your application by fetching related data only when necessary, minimizing unnecessary database queries, and reducing memory usage.


## Calculate Totals Using Conditional Aggregates

calculationg totals,

option one: $statuses = (object) [];
$statuses->requested = Feature::where('status', 'Requested')->count();
$statuses->planned = Feature::where('status', 'Planned')->count();
$statuses->completed = Feature::where('status', 'Completed')->count();

it will send 3 different queries to the database as "select coun(*) as aggregate from "features" where "status" = "Requested"


the right way:
$statuses = Feature::toBase() -- //toBase, we do not wanna feature model return back, instead we want a collection of different totals

$statuses = Feature::toBase()
            ->selectRaw("count(case when status = 'Requested' then 1 end) as requested")
            ->selectRaw("count(case when status = 'Planned' then 1 end) as planned")
            ->selectRaw("count(case when status = 'Completed' then 1 end) as completed")
            ->first();
            
it will send single query. as
"select 
   count(case when status = 'Requested' then 1 end), 
   count(case when status = 'Planned' then 1 end), 
   count(case when status = 'Completed' then 1 end)
from features"

## Optimizing Circular Relationships


