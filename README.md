# never-null-mutex
a mutex with non-zero default-, and implicit uninitialized-state. I herewith consider this idea public and attributable to degski, he/her with access to this repo.


### desciption
in [the art of] programming for concurrency, the use of atomic flags (last member to be constructed) is mandated. a non-zero atomic_flag constructed (with value=1) indicates construction-finalized of the specific object under construction (atomically). iff we use a non-zero mutex as the last object member, the mutex itself can stand-in as **construction-finalized-flag** while the object is under construction. after construction, the underlying `std::atomic` is repurposed as a mutex, and allows for fine-grained locking, at the lowest possible level, at **'zero'** cost (as the flag is a requirement, a 'must have'). the uninitialized state is implicit by not being any of the possible valid states, including **unlocked** (state = 1). a read-write-spinlock, can be designed at zero-extra-cost as compared to a write-spinlock of the same design.


an example of `never_null_mutex` (a test-test-and-swap spinlock) can be implemented like so:


    template<typename FlagType = char>
    struct never_null_mutex final {

        never_null_mutex ( ) noexcept                     = default;
        never_null_mutex ( never_null_mutex const & )     = delete;
        never_null_mutex ( never_null_mutex && ) noexcept = delete;
        ~never_null_mutex ( ) noexcept                    = default;

        never_null_mutex & operator= ( never_null_mutex const & )     = delete;
        never_null_mutex & operator= ( never_null_mutex && ) noexcept = delete;

        static constexpr int uninitialized               = 0;
        static constexpr int unlocked                    = 1;
        static constexpr int locked_reader               = 2;
        static constexpr int unlocked_locked_reader_mask = 3;
        static constexpr int locked_writer               = 4;

        ALWAYS_INLINE void lock ( ) noexcept {
            do {
                while ( unlocked != flag )
                    yield ( );
            } while ( not try_lock ( ) );
        }
        
        [[nodiscard]] ALWAYS_INLINE bool try_lock ( ) noexcept {
            return unlocked == flag.exchange ( locked_writer, std::memory_order_acquire );
        }
        
        ALWAYS_INLINE void unlock ( ) noexcept { flag.store ( unlocked, std::memory_order_release ); }

        ALWAYS_INLINE void lock ( ) const noexcept {
            do {
                while ( not( unlocked_locked_reader_mask & flag ) )
                    yield ( );
            } while ( not try_lock ( ) );
        }
        
        [[nodiscard]] ALWAYS_INLINE bool try_lock ( ) const noexcept {
            return unlocked_locked_reader_mask &
                   const_cast<std::atomic<int> *> ( &flag )->exchange ( locked_reader, std::memory_order_acquire );
        }
        
        ALWAYS_INLINE void unlock ( ) const noexcept {
            const_cast<std::atomic<FlagType> *> ( &flag )->store ( unlocked, std::memory_order_release );
        }

        private:
        std::atomic<int> flag = { unlocked };
    };

