# Expression modifier
There's an interesting paradigm in the Solidity compiler called `modifier`.  I
find it interestring and tried to see how it can be apply to C.

## Consideration
Consider a spinlock `lck` that you have acquire in order to access some datum `data`.

### Classic approach
This is the classic thing to do.  Using GCC atomic primitives.

```C
typedef _Bool spinlock_t;

static inline void atomic_acquire_spinlock(spinlock_t *lock)
{
        /* Burn CPU */
        while (__atomic_test_and_set(lock, __ATOMIC_ACQUIRE)) {
        }
}

static inline void atomic_release_spinlock(spinlock_t *lock)
{
        __atomic_clear(lock, __ATOMIC_RELEASE);
}

static spinlock_t lck;

void *my_thread(void *data)
{
        atomic_acquire_spinlock(&lck);
        {
            printf("%s\n", (char*)data);
        }
        atomic_release_spinlock(&lck);
}
```

### Hacky approach
I developed this approach to avoid having to do the acquire/release pair.

```C
typedef _Bool spinlock_t;

static inline _Bool atomic_acquire_spinlock(spinlock_t *lock)
{
	while (__atomic_test_and_set(lock, __ATOMIC_ACQUIRE)) {
		/* BURN CPU */
	}

	return true;
}

static inline _Bool atomic_release_spinlock(spinlock_t *lock)
{
	__atomic_clear(lock, __ATOMIC_RELEASE);

	return false;
}

#define CAT_PRIMITIVE(X, Y) X ## Y
#define CAT(X, Y) CAT_PRIMITIVE(X, Y)

#define MAKE_ID(PREFIX) CAT(PREFIX, __COUNTER__)

#define WITH_CONTEXT_PRIMITIVE(VAR, IN, OUT, ARGS...) for (_Bool VAR=IN(ARGS); VAR; VAR=OUT(ARGS))
#define WITH_CONTEXT(ARGS...) WITH_CONTEXT_PRIMITIVE(MAKE_ID(atomic_once), ARGS)

#define with_spinlock(LOCK) WITH_CONTEXT(atomic_acquire_spinlock, atomic_release_spinlock, LOCK)

static spinlock_t lck;

void *my_thread(void *data)
{
        with_spinlock(&lck) {
            printf("%s\n", (char*)data);
        }
}
```

### Modifier approach

Having a modifier that can take the next expression and modify it would yield
something like this

```C
typedef _Bool spinlock_t;

modifier with_spinlock(spintlock_t *lock)
{
	/* Burn CPU until acquire */
	while (__atomic_test_and_set(lock, __ATOMIC_ACQUIRE)) {
	}

	/* Emit expression here */
	@;

	/* Release */
	__atomic_clear(lock, __ATOMIC_RELEASE);
}

static spinlock_t lck;

void *my_thread(void *data)
{
        with_spinlock(&lck) {
            printf("%s\n", (char*)data);
        }
}

/* Will generate something like this */
static inline 
void *my_thread(void *data)
{
    {
		while (__atomic_test_and_set(&lck, __ATOMIC_ACQUIRE)) {
	    }

		/* Emitted expression */
		printf("%s\n", (char*)data);

		__atomic_clear(&lck, __ATOMIC_RELEASE);
    }
}
```
