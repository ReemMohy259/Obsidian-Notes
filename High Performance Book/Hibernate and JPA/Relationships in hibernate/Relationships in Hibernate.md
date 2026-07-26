# There are 5 types of Relationships

### 1. @ManyToOne

#### Definition
`@ManyToOne` added in the child side to reference the parent through the foreign key

Where the name "**post_id**" refences to the foreign key column name, also the table that has this annotation is the child but it also is the **owner of the relationship**

```java
@ManyToOne 
@JoinColumn(name = "post_id")
private Post post;
```
---
### 2. @OneToMany

#### Definition
`@OneToMany` added in the parent side to get all the children for it

#### Bidirectional @OneToMany

The `mappedBy` must be set with the name of the field that has the `@ManyToOne` in the child entity.

The `cascade` lets you define what will happen to the child entities when the parent gets **inserted** or **deleted**.

The `orphanRemoval` when this is set to **true**, it will delete the child entity when the foreign key column is set to **null** in the child entity.

```java
@OneToMany(mappedBy = "post", cascade = CascadeType.ALL, orphanRemoval = true)
private List<PostComment> comments = new ArrayList<>();
```

Since the parent is the **inverse end** in the relationship (not the owner), it needs to **use synchronization methods** to synchronize the child entity with the parent entity in the database.

```java
public void addComment(PostComment comment) {  
    comments.add(comment);  
    comment.setPost(this);  
}  
  
public void removeComment(PostComment comment) {  
    comments.remove(comment);  
    comment.setPost(null);  
}
```

##### Example:

```java
Post post = new Post("First post");
 
PostComment comment1 = new PostComment("My first review"); post.addComment(comment1); 
PostComment comment2 = new PostComment("My second review"); post.addComment(comment2); 

entityManager.persist(post);
```

resulting query:

```java
INSERT INTO post (title, id) VALUES ('First post', 1)

INSERT INTO post_comment (post_id, review, id) 
VALUES (1,'My first review', 2)

INSERT INTO post_comment (post_id, review, id) 
VALUES (1, 'My second review', 3)
```

Also in the example the `post_comment` rows got inserted because of the `cascade` type set.


> [!NOTE] 
> The bidirectional `@OneToMany` association generates efficient DML statements because the `@ManyToOne` mapping is in charge of the table relationship. Because it simplifies data access operations as well, the bidirectional `@OneToMany` association is worth considering **when the size of the child records is relatively low**.

##### The `java.lang.Object.equals`  method in Javadoc demands the strategy be:

- **Reflexive**  
    `x.equals(x)` must always be `true`.
- **Symmetric**  
    If `x.equals(y)` is `true`, then `y.equals(x)` must also be `true`.
- **Transitive**  
    If `x.equals(y)` is `true`, and `y.equals(z)` is `true`, then `x.equals(z)` must also be `true`.
- **Consistent**  
    - Repeated calls should return the same result **as long as the objects haven’t changed**.
	- but in hibernate it has be consistent throughout all the [[Entity State Transitions]] (e.g. new/transient, managed, detached, removed)

---
#### Unidirectional @OneToMany

If you use the `@OneToMany` annotation on its own without the child having the `@ManyToOne` annotation inside it, it will create a **junction table** that has the relation with the child entity so its mostly used in **bidirectional relationships**.

Even though this way is **simpler**, it is still much more **efficient** to the use **bidirectional** `@OneToMany`.

##### Using Plain `List`

``` java
@OneToMany(cascade = CascadeType.ALL, orphanRemoval = true) private List<PostComment> comments = new ArrayList<>();
```

![[Pasted image 20260506181440.png|697]]


This approach is **inefficient** for multiple reasons.

1. When inserting new records in there are **extra records inserted** due to the extra **junction table**
```java
Post post = new Post("First post");
post.getComments().add(new PostComment("My first review"));
post.getComments().add(new PostComment("My second review"));
post.getComments().add(new PostComment("My third review"));
```

```sql
INSERT INTO post (title, id) VALUES ('First post', 1) 
INSERT INTO post_comment (review, id) 
VALUES ('My first review', 2)
INSERT INTO post_comment (review, id) 
VALUES ('My second review', 3)
INSERT INTO post_comment (review, id) 
VALUES ('My third review', 4)
INSERT INTO post_post_comment (Post_id, comments_id) 
VALUES (1, 2)
INSERT INTO post_post_comment (Post_id, comments_id) 
VALUES (1, 3)
INSERT INTO post_post_comment (Post_id, comments_id) VALUES (1, 4)
```

2. When removing an element from the list it **deletes all the records** from the database and then **reinserts** the **ones remaining** in the list in memory.
```java
post.getComments().remove(0);
```

```sql
DELETE FROM post_post_comment WHERE Post_id = 1
INSERT INTO post_post_comment (Post_id, comments_id)
VALUES (1, 3)
INSERT INTO post_post_comment (Post_id, comments_id)
VALUES (1, 4) DELETE FROM post_comment WHERE id = 2
```

3. Reading data requires **2 joins** instead of **one**.

##### Using `List` With `@OrderColumn`

```java
@OneToMany(cascade = CascadeType.ALL, orphanRemoval = true) @OrderColumn(name = "entry")
private List<PostComment> comments = new ArrayList<>();
```

![[Pasted image 20260506184658.png]]

This is approach is **bit more efficient** but:

1. When adding elements to the database it still issues extra statements because of the **extra junction table**.

```java
post.getComments().add(new PostComment("My first review")); post.getComments().add(new PostComment("My second review")); post.getComments().add(new PostComment("My third review"));
```

```sql
INSERT INTO post_comment (review, id)
VALUES ('My first review', 2) 
INSERT INTO post_comment (review, id) 
VALUES ('My second review', 3) 
INSERT INTO post_comment (review, id)
VALUES ('My third review', 4)
INSERT INTO post_post_comment (Post_id, entry, comments_id) VALUES (1, 0, 2)
INSERT INTO post_post_comment (Post_id, entry, comments_id) VALUES (1, 1, 3) 
INSERT INTO post_post_comment (Post_id, entry, comments_id) VALUES (1, 2, 4)
```

2. When removing elements from the **tail of the collection** it works as **normal delete statements** with **no extra statements**.

```java
post.getComments().remove(2);
```

```sql
DELETE FROM post_post_comment WHERE Post_id = 1 AND entry = 2 DELETE FROM post_comment WHERE id = 4
```

3. But when trying to **remove elements** of the list, it will move all other entries to their correct positions to simulate the **list remove behavior**.
	The **closer** an element is to the **head of the list**, the **more update statements** must be issued.

```java
post.getComments().remove(0);
```

```sql
DELETE FROM post_post_comment WHERE Post_id = 1 AND entry = 2 UPDATE post_post_comment SET comments_id = 4 
WHERE Post_id = 1 AND entry = 1 
UPDATE post_post_comment SET comments_id = 3 
WHERE Post_id = 1 AND entry = 0
DELETE FROM post_comment WHERE id = 2
```

##### Using `List` With `@JoinColumn

This is the **most efficient** one, it will add a **new foreign key column** in the child entity in the database that **references the primary key** of the parent entity **without changing or adding new fields** in the entity class of the child (JPA knows about the column). 

The `@JoinColumn` annotation defines **customizations** for the foreign key column.

```java
@OneToMany(cascade = CascadeType.ALL, orphanRemoval = true) @JoinColumn(name = "post_id")
private List<PostComment> comments = new ArrayList<>();
```

1. When adding new elements, it **issues extra update statements** to set the **child's foreign key** to the **parent's primary key**.

```java
post.getComments().add(new PostComment("My first review")); post.getComments().add(new PostComment("My second review")); post.getComments().add(new PostComment("My third review")); 
```

```sql
INSERT INTO post_comment (review, id) 
VALUES ('My first review', 2)
INSERT INTO post_comment (review, id) 
VALUES ('My second review', 3)
INSERT INTO post_comment (review, id) 
VALUES ('My third review', 4)
UPDATE post_comment SET post_id = 1 WHERE id = 2
UPDATE post_comment SET post_id = 1 WHERE id = 3
UPDATE post_comment SET post_id = 1 WHERE id = 4
```

2. When trying remove elements it sets the **child's foreign key to null** so the `orphanRemoval` logic issues a **delete statement** and removes it, and it doesn't matter which element you are trying to remove.

```java
post.getComments().remove(2);
```

```sql
UPDATE post_comment SET post_id = null
WHERE post_id = 1 AND id = 4
DELETE from post_comment WHERE id = 4
```


> [!NOTE] Bidirectional @OneToMany with @JoinColumn relationship
> The `@OneToMany` with `@JoinColumn` association can also be turned into a **bidirectional relationship**, but it requires instructing the **child-side to avoid** **any** insert and update **synchronization**

```java
@ManyToOne @JoinColumn(name = "post_id", insertable = false, updatable = false) 
private Post post;
```

##### Using `Set`

This is more efficient than the list even the list with the `@JoinColumn`, the slug attribute is added to provide a way to make objects unique instead of a business key.

```java
@Entity(name = "PostComment")  
@Table(name = "post_comment")  
public class PostComment {  
    
    @Id  
    @GeneratedValue    
    private Long id;  
    private String slug;  
    private String review;
    
	@OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
	private Set<PostComment> comments = new HashSet<>();
    
    public PostComment() {  
        byte[] bytes = new byte[8];  
        ByteBuffer.wrap(bytes).putDouble(Math.random());  
        slug = Base64.getEncoder().encodeToString(bytes);  
    }  
  
    public PostComment(String review) {  
        this();  
        this.review = review;  
    } 
    
    //Getters and setters omitted for brevity   
    
    @Override  
    public boolean equals(Object o) {  
        if (this == o)
	        return true;
        if (o == null || getClass() != o.getClass())
		    return false;
        PostComment comment = (PostComment) o;
        return Objects.equals(slug, comment.getSlug());  
    }  
  
    @Override  
    public int hashCode() {
        return Objects.hash(slug);  
    }
}
```

1. When adding elements.

```java
post.getComments().add(new PostComment("My first review")); post.getComments().add(new PostComment("My second review")); post.getComments().add(new PostComment("My third review"));
```

```sql
INSERT INTO post_comment (review, slug, id)
VALUES ('My second review', 'P+HLCF25scI=', 2)
INSERT INTO post_comment (review, slug, id)
VALUES ('My first review', 'P9y8OGLTCyg=', 3) 
INSERT INTO post_comment (review, slug, id)
VALUES ('My third review', 'P+fWF+Ck/LY=', 4)

INSERT INTO post_post_comment (Post_id, comments_id) VALUES (1, 2)
INSERT INTO post_post_comment (Post_id, comments_id) VALUES (1, 3)
INSERT INTO post_post_comment (Post_id, comments_id) VALUES (1, 4)
```

2. The remove operation is much more efficient this time.

```java
for (PostComment comment: new ArrayList<>(post.getComments())) {      post.getComments().remove(comment); 
}
```

```sql
DELETE FROM post_post_comment WHERE Post_id = 1
DELETE FROM post_comment WHERE id = 2 
DELETE FROM post_comment WHERE id = 3 
DELETE FROM post_comment WHERE id = 4
```

To **avoid using a secondary table**, the `@OneToMany` mapping can use the `@JoinColumn` annotation.

```java
@OneToMany(cascade = CascadeType.ALL, orphanRemoval = true) @JoinColumn(name = "post_id")
private Set<PostComment> comments = new HashSet<>();
```

Upon **inserting** three $PostComment$ entities, Hibernate generates the following statements: 

```sql
INSERT INTO post_comment (review, slug, id) 
VALUES ('My third review', 'P8pcnLprqcQ=', 2)
INSERT INTO post_comment (review, slug, id) 
VALUES ('My second review', 'P+Gau1+Hhxs=', 3)
INSERT INTO post_comment (review, slug, id) 
VALUES ('My first review', 'P+kr9LOQTK0=', 4) 

UPDATE post_comment SET post_id = 1 WHERE id = 2 
UPDATE post_comment SET post_id = 1 WHERE id = 3 
UPDATE post_comment SET post_id = 1 WHERE id = 4
```

When **deleting** all three $PostComment$ entities, the generated statements look like this:

```sql
UPDATE post_comment SET post_id = null WHERE post_id = 1

DELETE FROM post_comment WHERE id = 2
DELETE FROM post_comment WHERE id = 3
DELETE FROM post_comment WHERE id = 4
```

---

### 3. @ElementCollection

#### Definition
`@ElementCollection` it is used to store elements of simple types or embeddable such as (`List<String>`,  `List<Object>`), that type is not an entity on its own.

![[Pasted image 20260506215140.png]]

![[Pasted image 20260506215200.png|314]]
#### `@ElementCollection` on List

```java
@ElementCollection 
private List comments = new ArrayList<>();
```

1. When it comes to **adding or removing child records**, the `@ElementCollection` behaves like a **unidirectional** `@OneToMany `relationship, annotated with `CascadeType.ALL` and `orphanRemoval`.

```java
post.getComments().add("My first review"); post.getComments().add("My second review"); post.getComments().add("My third review");
```

```sql
INSERT INTO Post_comments (Post_id, comments)
VALUES (1, 'My first review')
INSERT INTO Post_comments (Post_id, comments)
VALUES (1, 'My second review')
INSERT INTO Post_comments (Post_id, comments)
VALUES (1, 'My third review')
```

2. The remove operation uses the same logic as the **unidirectional** `@OneToMany` association, hibernate deletes all the associated child-side records and re-inserts the in-memory ones back into the database table.

```java
post.getComments().remove(0);
```

```sql
DELETE FROM Post_comments WHERE Post_id = 1 
INSERT INTO Post_comments (Post_id, comments)
VALUES (1, 'My second review')
INSERT INTO Post_comments (Post_id, comments)
VALUES (1, 'My third review')
```


> [!NOTE] 
> Using an `@ElementCollection` with a List is **not very efficient** because the association is considered to be a **bag**, in Hibernate terminology. 
> 
>A bag **does not guarantee** that elements are **uniquely identifiable**, hence Hibernate needs to **delete and reinsert the elements** associated with a given parent entity whenever a **change occurs** to the `@ElementCollection` List.

#### `@ElementCollection` on Set

```java
@ElementCollection
private Set comments = new HashSet<>();
```

1. Inserting here is the same as the list
2. When removing elements from the set
```java
post.getComments().remove("My first review");
```

```sql
DELETE FROM Post_comments WHERE Post_id = 1 and 
comments = 'My first review'
```


> [!NOTE]
> If you are going to use the `@ElementCollection` its better to use the set since it is much more efficient, but it’s still much better if you replace the `@ElementCollection` with either a bidirectional `@OneToMany` association or just a simple `@ManyToOne` association
> 
> This way, you can **remove elements without having to fetch the collection** via the parent entity.


---

### 4. @OneToOne

#### Definition
From a **database perspective**, the **one-to-one** association is based on a **foreign key** that is constrained to be **unique**. This way, a **parent row** can be referenced by **at most one child** record only.

#### Unidirectional @OneToOne

![[Pasted image 20260508164847.png]]

![[Pasted image 20260508164946.png]]

```java
@OneToOne 
@JoinColumn(name = "post_id")
private Post post;
```

The unidirectional `@OneToOne` association **controls** the associated **foreign key**, so, when the post attribute is set:

```java
Post post = entityManager.find(Post.class, 1L);
PostDetails details = new PostDetails("John Doe"); details.setPost(post);
entityManager.persist(details);
```

Hibernate populate the foreign key column with the associated post identifier:

```sql
INSERT INTO post_details (created_by, created_on, post_id, id) VALUES ('John Doe', '2016-01-08 11:28:21.317', 1, 2)
```

Even if this is a **unidirectional association**, the `Post` entity is still the **parent-side** of this relationship. To fetch the associated `PostDetails`, a JPQL query is needed:

```java
PostDetails details = entityManager.createQuery( "select pd " + "from PostDetails pd " +
"where pd.post = :post", PostDetails.class)
.setParameter("post", post)
.getSingleResult();
```

If the `Post` entity always needs its `PostDetails`, a **separate query might not be desirable**. To overcome this **limitation**, it is important to know the `PostDetails` **identifier prior to loading** the entity.

Making the relation like this tells **hibernate** and the **database** that the `Post` is not only the **foreign key** its **also the primary key** of the table and **removes the extra foreign key column**, which **improves performance** and **reduces memory** usage, this is also the preferred approach.
```java
@OneToOne
@MapsId
private Post post;
```

![[Pasted image 20260508170317.png]]

Because `PostDetails` has the same identifier as the parent Post entity, it can be fetched without having to write a **JPQL** query.

```java
PostDetails details = entityManager.find(PostDetails.class, post.getId());
```

#### Bidirectional @OneToOne

A bidirectional `@OneToOne` association allows the **parent entity** to **map** the **child-side** as well:

![[Pasted image 20260509003441.png]]

The parent-side defines a `mappedBy` attribute because the child-side is still in charge of this JPA relationship:

```java
@OneToOne(mappedBy = "post", cascade = CascadeType.ALL,
fetch = FetchType.LAZY)
private PostDetails details;
```

Since this is a bidirectional realtionship we need syncroization method.

```java
public void setDetails(PostDetails details) {  
    if (details == null) {  
        if (this.details != null) this.details.setPost(null);  
    } else details.setPost(this);  
    this.details = details;  
}
```

Unlike `@OneToMany`, where **Hibernate can assign an empty proxy collection**, a parent-side `@OneToOne` must determine whether the child reference is `null` or an **actual entity/proxy**.  

Since **only the child side** owns the **foreign key**, the parent side requires an **extra query to check** whether a corresponding **child row exists** even if the `fetch` is set to lazy.

```java
Post post = entityManager.find(Post.class, 1L);
```

Hibernate fetches the child entity as well, so, instead of only one query, Hibernate requires two select statements:

Which can cause **performance issues** if you don't need to fetch the child entity you still get that extra **sql statement**.
```sql
SELECT p.id AS id1_0_0_, p.title AS title2_0_0_ FROM post p WHERE p.id = 1 

SELECT pd.post_id AS post_id3_1_0_, pd.created_by AS created_1_1_0_, pd.created_on AS created_2_1_0_ FROM post_details pd WHERE pd.post_id = 1
```

---
### 5. @ManyToMany


![[Pasted image 20260509004656.png]]

#### Unidirectional `@ManyToMany` List

We map it from the `Post` side since it will be used the most.

![[Pasted image 20260509014335.png]]

```java
@ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE}) @JoinTable(name = "post_tag", 
	joinColumns = @JoinColumn(name = "post_id"), 
	inverseJoinColumns = @JoinColumn(name = "tag_id")
) 
private List<Tag> tags = new ArrayList<>();
```

```java
Post post1 = new Post("JPA with Hibernate");
Post post2 = new Post("Native Hibernate");

Tag tag1 = new Tag("Java");
Tag tag2 = new Tag("Hibernate");

post1.getTags().add(tag1);
post1.getTags().add(tag2);
post2.getTags().add(tag1);

entityManager.persist(post1);
entityManager.persist(post2);
```

```sql
INSERT INTO post (title, id) VALUES ('JPA with Hibernate', 1) INSERT INTO post (title, id) VALUES ('Native Hibernate', 4)

INSERT INTO tag (name, id) VALUES ('Java', 2)
INSERT INTO tag (name, id) VALUES ('Hibernate', 3)

INSERT INTO post_tag (post_id, tag_id) VALUES (1, 2)
INSERT INTO post_tag (post_id, tag_id) VALUES (1, 3)
INSERT INTO post_tag (post_id, tag_id) VALUES (4, 2)
```

For `@ManyToMany` associations, `CascadeType.REMOVE` **does not make too much sense** when both sides represent **independent entities**. In this case, removing a `Post` entity should not trigger a `Tag` removal because the `Tag` can be referenced by other posts as well. The same arguments apply to **orphan removal** since removing an entry from the tags collection should only delete the junction record and not the target Tag entity.

> [!NOTE]
> For both **unidirectional** and **bidirectional** associations, it is better to avoid the `CascadeType.REMOVE` mapping. Instead of `CascadeType.ALL`, the cascade attributes should be declared explicitly (e.g. `CascadeType.PERSIST`, `CascadeType.MERGE`).

```java
post1.getTags().remove(tag1);
```

```sql
DELETE FROM post_tag WHERE post_id = 1
INSERT INTO post_tag (post_id, tag_id) VALUES (1, 3)
```

#### Unidirectional @ManyToMany Set

```java
@ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE}) @JoinTable(name = "post_tag", 
	joinColumns = @JoinColumn(name = "post_id"),                      inverseJoinColumns = @JoinColumn(name = "tag_id") 
) 
private Set<Tag> tags = new HashSet<>();
```

```java
post1.getTags().remove(tag1);
```

```sql
DELETE FROM post_tag WHERE post_id = 1 AND tag_id = 2
```

For a `@ManyToMany` association, it’s better to use the `Set` collection type, instead of `List`.

#### Bidirectional @ManyToMany

![[Pasted image 20260509022042.png]]

Because the **junction table** is hidden when using the default `@ManyToMany` mapping, the **application developer** must choose an owning and a `mappedBy` side.

The side that has the `mappedBy` defines the **inverse end** (**not the owner**).

```java
@ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE}) @JoinTable(name = "post_tag", 
	joinColumns = @JoinColumn(name = "post_id"),                      inverseJoinColumns = @JoinColumn(name = "tag_id") 
) 
private Set<Tag> tags = new HashSet<>();
```

```java
@ManyToMany(mappedBy = "tags") 
private Set<Post> posts = new HashSet<>();
```

```java
public void addTag(Tag tag) {
    tags.add(tag);
    tag.getPosts().add(this);
}

public void removeTag(Tag tag) {
    tags.remove(tag);  
    tag.getPosts().remove(this);
}
```

#### The @OneToMany alternative

The `PostTagId` maps the `postId` and `tagId` foreign key columns, and the `PostTag` entity uses the `postId` and `tagId` properties of the `PostTagId` identifier to map the post and the tag `@ManyToOne` associations.

![[Pasted image 20260509031047.png]]

The Post entity maps the **bidirectional** `@OneToMany` side of the post `@ManyToOne` association from the `PostTag` child entity:

```java
@OneToMany(mappedBy = "post", cascade = CascadeType.ALL, orphanRemoval = true) 
private List<PostTag> tags = new ArrayList<>();
```

The Tag entity maps the **bidirectional** `@OneToMany` side of the tag `@ManyToOne` association defined by the `PostTag` child entity:

```java
@OneToMany(mappedBy = "tag", cascade = CascadeType.ALL, orphanRemoval = true) 
private List<PostTag> posts = new ArrayList<>();
```

This way, the **bidirectional** `@ManyToMany` relationship is transformed in two bidirectional `@OneToMany` associations.

```java
@Embeddable  
public class PostTagId implements Serializable {  
    @Column(name = "post_id")
    private Long postId;
    @Column(name = "tag_id")
    private Long tagId;
  
    public PostTagId() {
    }  
    
    public PostTagId(Long postId, Long tagId) {
        this.postId = postId;  
        this.tagId = tagId;  
    }
  
    public Long getPostId() {
        return postId;  
    }
  
    public Long getTagId() {
        return tagId;  
    }
  
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;  
        if (o == null || getClass() != o.getClass()) 
		        return false;
        PostTagId that = (PostTagId) o;
        
        return Objects.equals(postId, that.getPostId() &&  
                Objects.equals(tagId, that.getTagId());  
    }  

    @Override  
    public int hashCode() {  
        return Objects.hash(postId, tagId);  
    }  
}
```

```java
@Entity 
@Table(name = "post_tag")
public class PostTag {
    @EmbeddedId  
    private PostTagId id;  
    
    @ManyToOne  
    @MapsId("postId")  
    private Post post;  
    
    @ManyToOne  
    @MapsId("tagId")  
    private Tag tag;  
  
    private PostTag() {  
    }  
    
    public PostTag(Post post, Tag tag) {  
        this.post = post;  
        this.tag = tag;  
        this.id = new PostTagId(post.getId(), tag.getId());  
    }  
  
    //Getters and setters omitted for brevity  
    @Override  
    
    public boolean equals(Object o) {  
        if (this == o) return true;  
        if (o == null || getClass() != o.getClass()) 
	        return false;  
	        
        PostTag that = (PostTag) o;  
        
        return Objects.equals(post, that.getPost() &&  
                Objects.equals(tag, that.getTag());  
    }  
  
    @Override  
    public int hashCode() {  
        return Objects.hash(post, tag);  
    }  
}
```

In a **bidirectional association**, **helper methods** are needed to keep both `@ManyToOne` and `@OneToMany` sides **synchronized**.  

Unlike a normal bidirectional `@OneToMany`, `removeTag` is **more complex** because it must find and detach the specific `PostTag` linking both the current `Post` and the target `Tag`.

```java
public void removeTag(Tag tag) {
    for (Iterator<PostTag> iterator = tags.iterator(); iterator.hasNext();) {  
        PostTag postTag = iterator.next();  
        if (postTag.getPost().equals(this) && postTag.getTag().equals(tag)) {  
            iterator.remove();  
            postTag.getTag().getPosts().remove(postTag);  
            postTag.setPost(null);  
            postTag.setTag(null);  
            break;  
        }  
    }  
}
```

When trying to remove a tag from a post.
```java
post1.removeTag(tag1);
```

```sql
DELETE FROM post_tag WHERE post_id = 1 AND tag_id = 3
```

Because the `@ManyToOne` side manages the foreign key column changes, the internal collection state of the `@OneToMany` side is not taken into consideration unless we remove an entry.

For this reason, reversing the order of the tags elements has absolutely no effect:

```java
post1.getTags().sort( 
	Comparator.comparing( 
		(PostTag postTag) -> postTag.getId().getTagId() ) .reversed() 
		);
```

To **materialize the order of elements**, the `@OrderColumn` must be used instead:

```java
@OneToMany(mappedBy = "post", cascade = CascadeType.ALL, orphanRemoval = true)
@OrderColumn(name = "entry")
private List<PostTag> tags = new ArrayList<>();
```

With this mapping, the `post_tag` junction table includes an `entry` column that stores the collection order.  

So, when the element order changes, Hibernate updates the `entry` values to keep the `PostTag` ordering synchronized.

```java
UPDATE post_tag SET entry = 0 WHERE post_id = 1 AND tag_id = 4 UPDATE post_tag SET entry = 1 WHERE post_id = 1 AND tag_id = 3
```

Mapping the junction table as an entity lets us store extra fields (e.g., `created_on`, `created_by`) in the many-to-many relationship table. 

This is not possible when using the standard JPA `@ManyToMany` mapping directly.