# never-null-mutex
a mutex with non-zero default state and implicit uninitialized state, an idea, which I herewith consider public and attributable to degski, he/her with access to this repo.


### desciption
in programming for concurrency, the use of atomic flags (as last member) is mandated to indicate construction-finalized of objects under construction. iff we use a non-zero mutex as 'our' last member, the mutex itself stands in as construction-finalized-flag and is now ready for fine-grained locking, at the lowest possible level, at 'zero' cost.


this mutex can be implemented like so:


    template<typename FlagType = char>
    struct spin_rw_lock final {

        spin_rw_lock ( ) noexcept                 = default;
        spin_rw_lock ( spin_rw_lock const & )     = delete;
        spin_rw_lock ( spin_rw_lock && ) noexcept = delete;
        ~spin_rw_lock ( ) noexcept                = default;

        spin_rw_lock & operator= ( spin_rw_lock const & ) = delete;
        spin_rw_lock & operator= ( spin_rw_lock && ) noexcept = delete;

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

