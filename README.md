## Introduction

This repository contains a comparative, extendible benchmarking suite for C and C++ hash-table libraries.

The benchmarks test the speed of inserting keys, replacing keys, looking up existing keys, look up nonexisting keys, deleting existing keys, deleting nonexisting keys, and iteration.

The following is an example of one outputted graph:

<picture><img src="example_graph.svg" alt="Example graph"></picture>

Complete results can be found here: [20 000 000 keys](https://verstablebenchmarks.netlify.app/result_2024-01-17T21_16_42_20000000), [2 000 000 keys](https://verstablebenchmarks.netlify.app/https://verstablebenchmarks.netlify.app/result_2024-01-17t23_51_00_2000000), and [200 000 keys](https://verstablebenchmarks.netlify.app/result_2024-01-18T00_09_43_200000).

## Installing

1. Download or clone the repository.

2. Install [Boost](https://www.boost.org/) if necessary, or disable the `boost_unordered_flat_map` shim by editing `config.h`.

## Building

Using GCC, compile with `g++ -I. -std=c++20 -static -O3 -DNDEBUG -Wall -Wpedantic main.cpp` from the master directory.

## Running

Close background processes, lock your CPU's frequency, and then run the executable. Under the out-of-the-box configuration, the benchmarks take approximately an hour to complete on my AMD Ryzen 7 5800H with the CPU frequency locked at 90%.

The resulting graphs and heatmap are outputted as a HMTL file, and the raw data is outputted as a CSV file. Both these files are named with a GMT timestamp and placed in the current working directory.

The graphs are interactive. Hover over a label to highlight the associated plot, and click the label to toggle the plot's visibility. The graphs automatically scale to the visible plots.

## Configuring

Modify global settings, including the total key count, the measurement frequency, the maximum load factor, and the blueprints and shims (see below) enabled by editing `config.h`.

## Built-in tables

The following hash-table libraries are included out-of-the-box:

* TODO ONCE FINALIZED

## Built-in blueprints

A _blueprint_ is a combination of a key type, value type, hash function, comparison function, and key set against which to measure each table's performance.

Three blueprints are included out-of-the-box:

- The `uint32_uint32_murmur` blueprint (_32-bit integer key, 32-bit value_) tests how the tables perform when the hash function and key comparison function are cheap, traversing buckets is cheap (i.e. does not cause many cache misses), and moving keys and values is cheap. This blueprint disadvantages tables that store metadata in a separate array because doing so necessarily causes at least one extra cache miss per lookup.

 - The `uint64_struct448_murmur` blueprint (_64-bit integer key, 448-bit value_) tests how the tables perform when the hash function and key comparison function are cheap, traversing buckets is expensive (typically a cache miss per bucket), and moving keys and values is expensive. This blueprint disadvantages tables that do not store metadata in a separate array (or do but access the buckets array with every probe anyway to check the key) and that move elements around often (e.g. Robin Hood).

- The `cstring_uint64_fnv1a` (_16-char c-string key, 64-bit value_) blueprint tests how the tables perform when the hash function and key comparison function are expensive. This blueprint disadvantages tables that lack a (metadata) mechanism to avoid most key comparisons or that rehash existing keys often.

## Adding a new table (via a shim)

Each hash-table library plugs into the benchmarks via a custom _shim_ that provides a standard API for basic hash-table operations.

To add a new table, follow these steps:

1. Create a directory, with your chosen name for the shim, in the `shims` directory and, ideally, install the hash-table library's headers there.

2. Create a file name `shim.h` in the new directory.

3. Insert the name of the shim to an empty shim slot in `config.h`.

4. In `shim.h`, define a struct template with the name of the shim (`new_shim` in the below documentation) and that satisfies the following requirements:

    `new_shim< void >` should contain these members:

    ```c++
    static constexpr const char *label =
      // A string literal containing the label of the table to appear in the outputted graphs.
    ;
 
    static constexpr const char *color =
      // A string literal containing the color of the table's label and plot to appear in the outputted
      //graphs, e.g. rgb( 255, 0, 0 ).
    ;
    ```

    For each blueprint, `new_shim< blueprint >` should contain the following members, where `blueprint` is the name of the blueprint, `table_type` is the table type for that blueprint, and `itr_type` is the type of the associated iterator:

    ```c++
    static table_type create_table()
    {
      // Returns an initialized instance of a table that uses blueprint::hash_key as the hash function,
      // blueprint::cmpr_keys as the comparison function, and MAX_LOAD_FACTOR as the maximum load
      // factor.
    }
    
    static void insert( table_type &table, const blueprint::key_type &key )
    {
      // Inserts the specified key, along with a dummy value, into the table, replacing any matching
      // key already in the table if it exists.
    }

    static void erase( table_type &table, const blueprint::key_type &key )
    {
      // Erases the specified key and associated value from the table, if the key exists.
    }
    
    static itr_type find( table_type &table, const blueprint::key_type &key )
    {
      // Returns an iterator to the specified key and associated value, if the key exists,
      // or an iterator indicating a nonexisting key (e.g. an end iterator, for tables that follow the
      // std::unordered_table API), if the key does not exist.
    }
    
    static itr_type begin_itr( table_type &table )
    {
      // Returns an iterator to the first key and associated value in the table, or an iterator
      // indicating a nonexisting key if the table is empty.
    }
    
    static bool is_itr_valid( table_type &table, itr_type &itr )
    {
      // Returns true if the specified iterator points to a key inside the table.
    }

    static void increment_itr( table_type &table, itr_type &itr )
    {
      // Increments the specified iterator to point to the next key in the table, or an iterator
      // indicating a nonexisting key if key to which the iterator currently points is the last one
      // in the table.
    }
    
    static blueprint::key_type &get_key_from_itr( table_type &table, itr_type &itr )
    {
      // Returns a reference to the key pointed to by the specified iterator.
    }
    
    static blueprint::value_type &get_value_from_itr( table_type &table, itr_type &itr )
    {
      // Returns a reference to the value pointed to by the specified iterator.
    }
    
    static void destroy_table( table_type &table )
    {
      // Frees all memory associated with the table.
      // For C++ tables, if freeing memory is handled by table_type's destructor, this function can be
      // left empty.
    }
    ```

  Refer to the built-in shims for examples of how a shim can be implemented for C++ tables that follow the `std::unordered_map` API and for C tables, which typically require explicit template specializations for each blueprint.

## Adding a new blueprint

To add a new blueprint, follow these steps:

1. Create a directory with your chosen name for the blueprint in the `blueprint` directory.

2. Create a file name `blueprint.h` in the new directory.

3. Insert the name of the blueprint into an empty blueprint slot in `config.h`.

4. In `blueprint.h`, define a struct with the name of the blueprint that contains the following members:

    ```c++
    static constexpr const char *label =
      // A string literal containing the label of the blueprint toappear in the outputted graphs.
    ;
    
    using key_type =
      // The type of the keys that the tables should contain.
    ;
    
    using value_type =
      // The value associated with each key.
    ;
    
    static uint64_t hash_key( const key_type &key )
    {
      // Returns the hash code of the specified key.
    }
    
    bool cmpr_keys( const key_type &key_1, const key_type &key_2 )
    {
      // Returns true if the two specified keys are equal.
    }
    
    void fill_unique_keys( std::vector< blueprint::key_type > &keys )
    {
      // Fills the specified vector with a set of unique keys.
      // The size vector is already initialized with the size KEY_COUNT + KEY_COUNT /
      // KEY_COUNT_MEASUREMENT_INTERVAL * 1000.
      // The order is unimportant because the keys will be shuffled.
      // This function is called exactly once.
    }
    
    ```

    `blueprint.h` should also declare a suitably named macro (e.g. `NEW_SHIM_ENABLED`) that allows each shim to use the preprocessor to optionally exclude a specialization for the blueprint.

5. Ensure that each shim supports the new blueprint struct as a template argument (either via the base template or via an explicit template specialization).

## Sharing results

The graphs embedded in the outputted HTML file are self-contained SVGs, so they can be trivially extracted and turned into standalone SVG files or embedded into other HTML files. They can even be used in GitHub READMEs (via `<img src="graph.svg">`), albeit without the interactive features.
