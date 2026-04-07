---
title: "Vibe Programming Quality in C++: Engineer vs. Non-Engineer Instructions to AI"
date: 2026-04-06
description: "A case study on cache system implementation through the lens of software engineering discipline — why the bottleneck in AI-assisted coding is no longer writing code, but knowing how to think about the problem."
tags: ["c++", "ai", "vibe-coding", "software-engineering", "llm"]
categories: ["programming"]
---

> **Topic:** Cache system (LRU, TTL, thread safety, metrics)  
> **Tool:** Cursor / Claude Code (AI-assisted live programming)  
> **Language:** C++17, STL only  
> **Focus:** Prompt quality & iterative refinement

## Introduction

Vibe programming is an emerging practice where the developer, or non-developer, describes in natural language what they want, and a tool such as Cursor or Claude Code generates the corresponding code. The quality of the final result depends almost entirely on the initial prompt and on how effectively the user iterates on it.

This creates an interesting asymmetry: **the bottleneck is no longer knowing how to write code, but knowing how to think about the problem.** That difference becomes brutally visible when we compare how a software engineer and someone without technical training ask the same thing from the AI.

The case study in this post is a cache system in C++. It is the ideal scenario for this comparison because it contains technical dimensions that are completely invisible to a non-developer: eviction policy, thread safety, TTL, load factor, observability. An engineer mentions all of these in the first prompt. A non-developer does not even know they exist.

---

## The Initial Prompt — The Most Revealing Moment

The first prompt someone writes tells the AI everything that person *does not know they need to ask for*. Below are the two prompts as they would appear in Cursor or Claude Code.

### Non-developer prompt

```
I need a cache in C++. It should store strings and retrieve them
by a key. It should be fast.
```

### Software engineer prompt

```
Implement an LRU cache in C++17 with the following:
- Configurable max capacity (number of entries)
- Per-entry TTL (automatic expiration in seconds)
- Thread-safe with per-operation locking granularity
- Strict LRU eviction using doubly-linked list + unordered_map for O(1) get/put
- Public interface: get(key)->optional<string>, put(key,value,ttl_seconds), invalidate(key), size(), clear()
- Basic metrics: hit rate, miss rate, eviction count
- No external dependencies, STL only
```

### The gap

The difference is not that the engineer writes more. It is that the engineer has already *designed the system in their head* before writing a single character. They know that "fast" is not a specification — they know to say O(1). They know that in production there will be multiple threads. They know that data becomes stale. They know that without metrics, the cache is a black box.

| What the non-developer sees | What the engineer sees |
|---|---|
| Store data | O(1) get/put |
| Retrieve data | Concurrency |
| Fast | TTL expiration |
| | Eviction policy |
| | Metrics |
| | No external dependencies |

---

## The Generated Code — Direct Comparison

### Result from the non-developer prompt

The AI, given a vague specification, makes reasonable assumptions and delivers the simplest possible implementation:

```cpp
#include <iostream>
#include <unordered_map>
#include <string>

class Cache {
    std::unordered_map<std::string, std::string> data_;

public:
    void set(const std::string& key, const std::string& value) {
        data_[key] = value;
    }

    std::string get(const std::string& key) {
        auto it = data_.find(key);
        if (it != data_.end()) return it->second;
        return ""; // empty string = "not found"?
                   // or was the stored value ""? Ambiguous.
    }

    void remove(const std::string& key) {
        data_.erase(key);
    }
};

int main() {
    Cache c;
    c.set("user:1", "Alice");
    std::cout << c.get("user:1")  << '\n'; // Alice
    std::cout << c.get("user:99") << '\n'; // "" — silent, ambiguous
}
```

This code compiles and "works" for the `main` example. But in production:

- It grows without bound until RAM is exhausted.
- On a multi-threaded server it produces data races and undefined behavior.
- There is no way to know whether a key expired, never existed, or whether its value was `""`.
- There is no observability: how many entries? How many hits vs. misses?

### Result from the engineer's prompt

The AI, given a precise specification, generates a coherent system:

```cpp
#include <unordered_map>
#include <list>
#include <optional>
#include <mutex>
#include <chrono>
#include <string>
#include <atomic>

class LRUCache {
    using Clock     = std::chrono::steady_clock;
    using TimePoint = Clock::time_point;

    struct Entry {
        std::string value;
        TimePoint   expires_at;
    };

    using ListIt = std::list<std::string>::iterator;

    const std::size_t          capacity_;
    std::list<std::string>     lru_order_;
    std::unordered_map<
        std::string,
        std::pair<Entry, ListIt>> map_;
    mutable std::mutex         mu_;

    // Atomics: readable without holding the lock
    std::atomic<uint64_t> hits_      {0};
    std::atomic<uint64_t> misses_    {0};
    std::atomic<uint64_t> evictions_ {0};

    bool is_expired(const Entry& e) const {
        return Clock::now() >= e.expires_at;
    }

    // Must be called with lock held
    void evict_lru() {
        if (lru_order_.empty()) return;
        map_.erase(lru_order_.back());
        lru_order_.pop_back();
        ++evictions_;
    }

public:
    explicit LRUCache(std::size_t capacity) : capacity_(capacity) {}

    // Non-copyable: mutex is non-copyable; copy semantics are ambiguous
    LRUCache(const LRUCache&)            = delete;
    LRUCache& operator=(const LRUCache&) = delete;

    std::optional<std::string> get(const std::string& key) {
        std::lock_guard lock(mu_);
        auto it = map_.find(key);
        if (it == map_.end()) { ++misses_; return std::nullopt; }

        auto& [entry, list_it] = it->second;
        if (is_expired(entry)) {
            lru_order_.erase(list_it);
            map_.erase(it);
            ++misses_;
            return std::nullopt;
        }
        // Promote to front -- this key is now the most recent
        lru_order_.splice(lru_order_.begin(), lru_order_, list_it);
        ++hits_;
        return entry.value;
    }

    void put(const std::string& key,
             const std::string& value,
             std::chrono::seconds ttl) {
        std::lock_guard lock(mu_);
        auto it = map_.find(key);
        if (it != map_.end()) {
            it->second.first = {value, Clock::now() + ttl};
            lru_order_.splice(
                lru_order_.begin(), lru_order_, it->second.second);
            return;
        }
        if (map_.size() >= capacity_) evict_lru();
        lru_order_.push_front(key);
        map_[key] = {{value, Clock::now() + ttl}, lru_order_.begin()};
    }

    bool invalidate(const std::string& key) {
        std::lock_guard lock(mu_);
        auto it = map_.find(key);
        if (it == map_.end()) return false;
        lru_order_.erase(it->second.second);
        map_.erase(it);
        return true;
    }

    void clear() {
        std::lock_guard lock(mu_);
        map_.clear();
        lru_order_.clear();
    }

    struct Stats {
        uint64_t hits, misses, evictions;
        double   hit_rate;
    };

    [[nodiscard]] Stats stats() const {
        uint64_t h = hits_, m = misses_;
        uint64_t total = h + m;
        return {h, m, evictions_,
                total ? double(h) / double(total) : 0.0};
    }

    [[nodiscard]] std::size_t size() const {
        std::lock_guard lock(mu_);
        return map_.size();
    }
};

int main() {
    using namespace std::chrono_literals;
    LRUCache cache(1000); // max 1000 entries

    cache.put("session:abc", "user_data_json", 300s);

    if (auto val = cache.get("session:abc")) {
        std::cout << *val << '\n';
    }

    auto s = cache.stats();
    std::cout << "Hit rate: " << s.hit_rate * 100 << "%\n";
}
```

---

## Dimensions Ignored by the Non-Developer Prompt

Every omission in the prompt maps to a production symptom:

| Dimension | Non-dev prompt | Production symptom | Engineer prompt |
|---|---|---|---|
| Eviction policy | Not mentioned | RAM grows to OOM | `"LRU strict"` |
| Thread safety | Not mentioned | Data race, crash, silent corruption | `"thread-safe, per-op lock"` |
| TTL/expiration | Not mentioned | Stale data served indefinitely | `"TTL per entry in seconds"` |
| Time complexity | *"fast"* | May be O(n) without awareness | `"O(1) get/put"` |
| Absence signal | Not mentioned | `""` = absent or empty value: ambiguous | `get -> optional<string>` |
| Observability | Not mentioned | No visibility in operation | `"hit rate, misses, evictions"` |
| Dependencies | Not mentioned | AI may introduce external libs | `"STL only, no deps"` |
| Max capacity | Not mentioned | No limit → no eviction → not really a cache | `"configurable capacity"` |

---

## Iteration — Where the Gap Widens

The initial prompt is only the beginning. What further separates the engineer from the non-developer is how each person *iterates* when the AI delivers something incomplete or incorrect.

**Non-developer (reactive):**
1. Iteration 1 → "Looks good, thanks!" — accepts the plain map
2. In production → **crash**: race condition under multi-threaded load
3. Late iteration → "Make it delete old entries too"
4. Outcome: accumulated patches, growing technical debt

**Software engineer (proactive):**
1. Iteration 1 → Reviews: is the lock per-operation?
2. Iteration 2 → "Switch to `shared_mutex` for read-heavy workloads"
3. Iteration 3 → "Add benchmark with 4 concurrent threads"
4. Outcome: validated system, ready to merge

The non-developer iterates *reactively*: problems are discovered when the system fails in production and patches are added one by one. Each iteration fixes the visible problem but leaves the hidden ones intact. The engineer iterates *proactively*: the generated code is reviewed against a mental model of the system, omissions are identified before they fail, and iterations are used to validate and refine rather than to fight fires.

---

## What the Generated Code Reflects About the Prompt

The AI does not invent what you did not ask for. It is deterministic in that sense: it fills gaps with the simplest possible choices. An `unordered_map` without a mutex is the correct answer to the prompt "store strings and retrieve them by key." The AI is not wrong — the prompt was incomplete.

This has an important consequence for teams adopting vibe programming tools: **code review becomes a review of the prompt, not of the code.** If the prompt does not mention thread safety, the generated code will not be thread-safe, and there is no visible warning to indicate this. The code compiles. Basic tests pass. The problem surfaces in production under concurrent load.

The engineer's value lies exactly there: they know what questions to ask the AI *before* the system fails.

---

## Implications for Teams and Organizations

Vibe programming does not democratize software engineering — it raises the floor but does not eliminate the ceiling. Anyone can now generate code that compiles. But the generated code inherits the invisible complexity of the domain, and understanding that complexity still requires technical training.

What changes is the distribution of work: **the engineer spends less time on syntax and more time on design, specification, and review.** Tools like Cursor or Claude Code are a multiplier, not a replacement. A senior engineer with Claude Code can produce in one day what previously took a week. A non-developer with Claude Code can produce in one day code that *appears* to work but carries a week's worth of bugs waiting to explode.

The right question for an organization is not "do we need fewer engineers now that we have AI?" but rather **"do we have engineers who know how to write the correct prompts?"**

---

## Summary

The pipeline from prompt to production looks very different depending on who is driving:

| Stage | Non-developer | Software engineer |
|---|---|---|
| Prompt | Vague, incomplete | Precise, designed |
| Generated code | Compiles, fails | Complete, safe |
| Production | Crash, data race | Stable, observable |

The cache case study makes it clear: the gap between an engineer and a non-developer using vibe programming is not in the code that comes out — it is in the mental model that goes in. The AI is an extremely capable translator from specification to code. The problem is that the non-developer's specification omits everything they cannot see because they do not know it exists: concurrency, expiration, eviction, asymptotic complexity, and observability.

In the context of tools like Cursor or Claude Code, the critical skill of a software engineer is no longer writing code; it is **writing prompts that encode years of production experience in a single instruction.** That is not something you learn from a vibe programming tutorial. You learn it by debugging a 3am crash that an unprotected `unordered_map` produces under concurrent load.
