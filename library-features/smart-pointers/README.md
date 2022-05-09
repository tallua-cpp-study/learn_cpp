# Smart Pointers

## Kind

1. `unique_ptr<T, Deleter>`
    1. own memory object
    2. helper function `make_unique<T>(_Args...)`
    3. you can assign custom deleter
    4. all constructor of unique_ptr are `noexcept`
    5. must use as rvalue reference for parameter
    6. usage:
    ```cpp
    struct Obj {
        Obj(std::string_view name)
            : name(name)
        {
            std::cout << "created: " << name << std::endl;
        }

        ~Obj()
        {
            std::cout << "destroyed: " << name << std::endl;
        }

        std::string name;
    };

    void test_unique_ptr() {
        auto obj1 = std::make_unique<Obj>("1"); // created: 1

        // ERROR: cannot copy
        //auto obj2 = obj1;

        {
            // you have to move to transfer ownership
            auto obj1_moved = std::move(obj1);
        }                                       // destroyed: 1
    }
    ```

2. `shared_ptr<T>`
    1. own memory with reference counter
    2. atomic copy/move
    3. helper function `make_shared<T>(_Args...)`
    4. may be cunstructed from `unique_ptr`
    5. usage:
    ```cpp
    struct Obj {
        Obj(std::string_view name)
            : name(name)
        {
            std::cout << "created: " << name << std::endl;
        }

        ~Obj()
        {
            std::cout << "destroyed: " << name << std::endl;
        }

        std::string name;
    };

    void test_shared_ptr() {
        auto obj1 = std::make_shared<Obj>("1");         // created: 1
        {
            std::shared_ptr<Obj> obj3_holder;
            {
                auto obj2 = std::make_shared<Obj>("2"); // created: 2
                auto obj3 = std::make_shared<Obj>("3"); // created: 3

                obj3_holder = obj3;
            }                                           // destroyed: 2
        }                                               // destroyed: 3
    }                                                   // destroyed: 1
    ```
    
    - possible implementation
    ```mermaid
    classDiagram

    class shared_ptr
    shared_ptr --> ControlBlock
    shared_ptr --> T
    
    class ControlBlock {
        <<atomic>>
        int shared_count;
        int weak_count;
    }
    
    class T {
        <<template type>>
    }
    ```

3. `weak_ptr<T>`
    1. doesn't own memory
    2. use `weak_ptr::lock()` to make it shared
       1. if shared_ptr already deleted, will return shared_ptr containing nullptr
       2. atomic operation
       3. `noexcept` operation
    3. atomic copy/move/lock
    4. usage:
    ```cpp
    struct Obj {
        Obj(std::string_view name)
            : name(name)
        {
            std::cout << "created: " << name << std::endl;
        }

        ~Obj()
        {
            std::cout << "destroyed: " << name << std::endl;
        }

        std::string name;
    };

    void test_weak_ptr() {
        auto obj1 = std::make_shared<Obj>("1");                 // created: 1
        {
            std::weak_ptr<Obj> obj2_weak;
            {
                auto obj2 = std::make_shared<Obj>("2");         // created: 2

                obj2_weak = obj2;
            }                                                   // destroyed: 2

            auto new_obj2 = obj2_weak.lock();                   // nullptr
            std::cout << (new_obj2 ? "t" : "f") << std::endl;   // f
        }
    }                                                           // destroyed: 1
    ```

4. `observer_ptr<T>`
    1. doesn't own memory
    2. experimental
    3. wrapper of raw pointer

---
## Notes

1. dereferencing smart pointers doesn't ensure null pointer
    ```cpp
    void undefined_behaviour() {
        std::shared_ptr<int> my_ptr(nullptr);

        std::cout << *my_ptr << std::endl;
    }
    ```

2. dereferencing `shared_ptr` doesn't ensure atomic
    ```cpp
    void race_condition() {
        auto my_counter = std::make_shared<int>(0);

        for (int i = 0; i < 10; ++i) {
            std::thread th([&]() {
                ++(*my_counter);
            });
            th.detach();
        }
    }
    ```

3. don't use as parameter if you don't need to transfer ownership
   1. use reference/raw pointer/`observer_ptr` instead

4. use `dynamic_pointer_cast<T>` or `static_pointer_cast<T>` for casting `shared_ptr`

5. `enable_shared_from_this<T>`
   1. use by inherit
   2. member function `shared_ptr<T> shared_from_this()`
      1. throw `bad_weak_ptr` exception if it is not `shared_ptr`

6. using `make_shared<T>` and `make_unique<T>` needs constructor/destuctor to be public

---
## Advantage
1. automatic memory management
2. declar ownership


## Disadvantage
1. overhead on `shared_ptr` copy
2. abusing `shared_ptr` may make cyclic reference
    ```cpp
    struct Book;
    struct Author;

    struct Book {
        shared_ptr<Author> author;
    };

    struct Author {
        shared_ptr<Book> book;
    };

    void cyclic_reference() {
        auto book = std::shared_ptr<Book>(new Book{});
        auto author = std::shared_ptr<Author>(new Author{});

        book->author = author;
        author->book = book;
    }
    ```
    - break by changing one shared_ptr to weak_ptr

---
## See also
1. [Reference](../../language-features/reference/README.md)
2. [Iterator](../iterators/README.md)

