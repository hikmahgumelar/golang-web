# Put A File

We put a file with code like this:

```go
func putFile(ctx context.Context, name string, rdr io.Reader) error {

	client, err := storage.NewClient(ctx)
	if err != nil {
		return err
	}
	defer client.Close()

	writer := client.Bucket(gcsBucket).Object(name).NewWriter(ctx)

	io.Copy(writer, rdr)
	// check for errors on io.Copy in production code!
	return writer.Close()
}
```

You can learn about the different types and methods in google/cloud by going to godoc.org and searching for `google/cloud` which will give you this link: [https://godoc.org/google.golang.org/cloud/storage](https://godoc.org/google.golang.org/cloud/storage)

My read through on the code would be like this:

In the `storage` package for Google cloud, you call `func newClient` which gives you a pointer to a client: `*storage.Client`.

Once you have a pointer to a client, there are two methods which you can call:

```go
func (c *Client) Bucket(name string) *BucketHandle
```

```go
func (c *Client) Close() error
```

Since there is a `Close()` method, this is a big hint that we should probably close a client. We can do this effectively with defer, as you see in the code sample above: `defer client.Close()`

To continue finding functionality in Google Cloud Storage, we will then also probably need to call `func (c *Client) Bucket(name string) *BucketHandle`. This makes sense as we will need to specify which bucket we want to access either to **put** or **get** a file.

So if we call `Bucket()` we are given a new type which is a pointer to a bucket handle: `*storage.BucketHandle`.

With a `*storage.BucketHandle`, we can see in the `storage` package documentation index that there are now several more methods available to us:
 
 ```go
 func (c *BucketHandle) ACL() *ACLHandle
 func (b *BucketHandle) Attrs(ctx context.Context) (*BucketAttrs, error)
 func (c *BucketHandle) DefaultObjectACL() *ACLHandle
 func (b *BucketHandle) List(ctx context.Context, q *Query) (*ObjectList, error)
 func (b *BucketHandle) Object(name string) *ObjectHandle
 ```
 
## ACL
 
This will let us control our **Access Control List**. Basically these are settings which we can set on our **bucket** to control who can access the **bucket** and what they can do. This is known as *scope* and *permissions*.
  
**Scope** defines who the permission applies to (for example, a specific user or group of users). Scopes are sometimes referred to as *grantees.* 

**Permissions** define the actions that can be performed against a bucket (for example, read or write).
   
We will also, later, be able to set ACL's for objects (files) which we store in Google Cloud Storage (GCS).
  
More on this later.
 
## Attrs
 
This will give us the attributes for a **bucket**. You can see the many different attributes at this link: [https://godoc.org/google.golang.org/cloud/storage#BucketAttrs](https://godoc.org/google.golang.org/cloud/storage#BucketAttrs) For convenience, I'm also listing them here:
 
```go
 type BucketAttrs struct {
     // Name is the name of the bucket.
     Name string
 
     // ACL is the list of access control rules on the bucket.
     ACL []ACLRule
 
     // DefaultObjectACL is the list of access controls to
     // apply to new objects when no object ACL is provided.
     DefaultObjectACL []ACLRule
 
     // Location is the location of the bucket. It defaults to "US".
     Location string
 
     // MetaGeneration is the metadata generation of the bucket.
     MetaGeneration int64
 
     // StorageClass is the storage class of the bucket. This defines
     // how objects in the bucket are stored and determines the SLA
     // and the cost of storage. Typical values are "STANDARD" and
     // "DURABLE_REDUCED_AVAILABILITY". Defaults to "STANDARD".
     StorageClass string
 
     // Created is the creation time of the bucket.
     Created time.Time
 }
 ```
 
We will also, later, be able to see Attrs for objects (files) which we store in Google Cloud Storage (GCS).
  
More on this later.
  
## DefaultObjectACL
  
This let's you set a default ACL which will be applied to newly created objects in this bucket that do not have a defined ACL.

## List

List lists objects from the bucket. You can specify a query to filter the results. If q is nil, no filtering is applied.

This is what we will use to query a bucket and have results returned.

```go
func (b *BucketHandle) List(ctx context.Context, q *Query) (*ObjectList, error)
```

## Object

This is perhaps the most commonly used method when working with a bucket.

Remember what we've done so far: We (1) got a Google Cloud Storage client, and then we (2) said that we wanted to work with a specific bucket, and now (3) we are going to say that we want to work with a specific object.

The code to do all of that is in our initial code sample up above. The excerpt of code to which I'm referring looks like this:


```go
 client.Bucket(gcsBucket).Object(name)
```

As we are learning how to **put** an object here, we will follow this thread of logic.

The `Object` method returns a pointer to an ObjectHandle `*storage.ObjectHandle`. You can see the func's signature here again:

```go
func (b *BucketHandle) Object(name string) *ObjectHandle
```

With a `*ObjectHandle` we once again have several methods available to us:
 
 ```go
 func (o *ObjectHandle) ACL() *ACLHandle
 func (o *ObjectHandle) Attrs(ctx context.Context) (*ObjectAttrs, error)
 func (o *ObjectHandle) CopyTo(ctx context.Context, dst *ObjectHandle, attrs *ObjectAttrs) (*ObjectAttrs, error)
 func (o *ObjectHandle) Delete(ctx context.Context) error
 func (o *ObjectHandle) NewRangeReader(ctx context.Context, offset, length int64) (*Reader, error)
 func (o *ObjectHandle) NewReader(ctx context.Context) (*Reader, error)
 func (o *ObjectHandle) NewWriter(ctx context.Context) *Writer
 func (o *ObjectHandle) Update(ctx context.Context, attrs ObjectAttrs) (*ObjectAttrs, error)
 func (o *ObjectHandle) WithConditions(conds ...Condition) *ObjectHandle
 ```

You can see that, for an **object** (and not the bucket), we can set the **ACL** (Access Control List) for an **object**, we can also see the **Attrs** for an object, we can use **CopyTo** to copy one object to another object, we can **Delete** an object, we can read an object with **NewReader** and we can write to an object with **NewWriter**.

We can also **Update** the attributes ( `ObjectAttrs` ) of an object. Please note that you *cannot* alter the actual binary of the object - there is no prepending or appending something to a video file, there is no adding binary to a picture, none of that. If you want to *change* the actual object, you need to *replace* the object with a new version. However, if you only want to *update* the *attributes* of an object, well then, you can use the **Update** method.
  
Regarding the last method, **WithConditions**, don't worry about this one right now. You can learn about it if, and when, you need it.

So to **put** an object, we're going to call **NewWriter** because, what we want to do, is write to this one particular object in this one particular bucket in Google Cloud Storage. We can see this from our initial code samply up above:

```go
writer := client.Bucket(gcsBucket).Object(name).NewWriter(ctx)
```

Now that we have a writer, we can use `io.Copy` to copy from something that implements the reader interface into our new object on GCS:

```go
	io.Copy(writer, rdr)
	// check for errors on io.Copy in production code!
	return writer.Close()
```

And, as you can see above, we should close our writer. We know this because, when we call `NewWriter` we are returned a pointer to a google cloud storage writer:

```go
func (o *ObjectHandle) NewWriter(ctx context.Context) *Writer
```

And with a `*storage.Writer`, we have these methods:
 
 ```go
  func (w *Writer) Attrs() *ObjectAttrs
  func (w *Writer) Close() error
  func (w *Writer) CloseWithError(err error) error
  func (w *Writer) Write(p []byte) (n int, err error)
 ```

You can see in the above methods that the type `storage.Writer` implements the `io.Writer` interface. Here is that interface from package `io`:

```go
type Writer interface {
    Write(p []byte) (n int, err error)
}
```

Since a `*storage.Writer` has a method with a signature `Write(p []byte) (n int, err error)` that matches the required method to implement the `io.Writer` interface, then the type `storage.Writer` implicitly impelements the writer interface.

That means we can use a `sotrage.Writer` in any func or method which asks for a `Writer`.

You can see this with `io.Copy`:

```go
func Copy(dst Writer, src Reader) (written int64, err error)
```

`io.Copy` is asking for a `Writer` so, since `storage.Writer` implements the writer interface, we can pass `io.Copy` a `storage.Writer`.

Ok. So that is a little bit about how you put an object in GCS, and a lot about reading documentation and understanding Go code.

Now onto how we get an object.

# Get

This is the code to get a file from Google Cloud Storage:

```go
func getFile(ctx context.Context, name string) (io.ReadCloser, error) {
	client, err := storage.NewClient(ctx)
	if err != nil {
		return nil, err
	}
	defer client.Close()

	return client.Bucket(gcsBucket).Object(name).NewReader(ctx)
}
```

The process is very similar to how we *put* an object.

We get a storage **client**, and then we specify which **bucket** we want to access, and then we specify which **object** (file) we want to access, and then we call the method `NewReader` which gives us back a pointer to type Reader from package storage `*storage.Reader` and with a `*storage.Reader` we then have all of these methods available to us:

```go
func (r *Reader) Close() error
func (r *Reader) ContentType() string
func (r *Reader) Read(p []byte) (int, error)
func (r *Reader) Remain() int64
func (r *Reader) Size() int64
```

So you can see we are going to need to *close* our reader. Why? Well, just imagine what would happen to your computer if you just kept opening files after files after files to read them yourself *and you never closed any of those files*. Eventually, obviously, your computer is going to freak out which is another way of saying that it will run out of resources, like memory, and then just crash and die a painful death. So, to avoid all of that, close your files if you open them.

You will also notice that there is this method:

```go
func (r *Reader) Read(p []byte) (int, error)
```

What that tells us is that the type `storage.Reader` (and please notice that I keep prefixing the type with the package from which it comes - this is something that a lot of people need to be shown, the notation is `package.Type`, you see, just like here, `storage.Reader` we're talking about the type `Reader` from the `storage` package, ok). Ok. Back to what I was saying. What that tells us is that the type `storage.Reader` implements the reader interface from package io:

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}
```

You see, package `io` has a type `Reader` which is an interface. To implement that interface you have to have this method `Read(p []byte) (n int, err error)`. If there is a type that has that method, then it implements the `Reader` interface. And any func or method which asks for a value of type `Reader` as an argument can now take **any** type which implements the `Reader` interface.
 
Again, all interfaces are implicitly implemented in Go. No need to explicitly say that something implements an interface. It automatically happens implicitly.

So the type which we just got, `storage.Reader`, implements the reader interface. That means we can now use things like:

**io.Copy**

```go
func Copy(dst Writer, src Reader) (written int64, err error)
```

**fmt.Fscan**
```go
func Fscan(r io.Reader, a ...interface{}) (n int, err error)
func Fscanf(r io.Reader, format string, a ...interface{}) (n int, err error)
func Fscanln(r io.Reader, a ...interface{}) (n int, err error)
```

**util.ReadAll**
```go
func ReadAll(r io.Reader) ([]byte, error)
```

**csv.NewReader**
```go
func NewReader(r io.Reader) *Reader
func (r *Reader) Read() (record []string, err error)
func (r *Reader) ReadAll() (records [][]string, err error)
```

**json.NewDecoder**
```go
type Decoder
func NewDecoder(r io.Reader) *Decoder
func (dec *Decoder) Buffered() io.Reader
func (dec *Decoder) Decode(v interface{}) error
func (dec *Decoder) More() bool
func (dec *Decoder) Token() (Token, error)
func (dec *Decoder) UseNumber()
```
