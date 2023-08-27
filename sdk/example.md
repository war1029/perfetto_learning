# 一、#include "trace_categories.h"
```c++
// The set of track event categories that the example is using.
PERFETTO_DEFINE_CATEGORIES(
    perfetto::Category("rendering")
        .SetDescription("Rendering and graphics events"),
    perfetto::Category("network.debug")
        .SetTags("debug")
        .SetDescription("Verbose network events"),
    perfetto::Category("audio.latency")
        .SetTags("verbose")
        .SetDescription("Detailed audio latency metrics"));
```
**设置监测类：**
```c++
// Register categories in the default (global) namespace. Warning: only one set
// of global categories can be defined in a single program. Create namespaced
// categories with PERFETTO_DEFINE_CATEGORIES_IN_NAMESPACE to work around this
// limitation.
#define PERFETTO_DEFINE_CATEGORIES(...)                           \
  PERFETTO_DEFINE_CATEGORIES_IN_NAMESPACE(perfetto, __VA_ARGS__); \
  PERFETTO_USE_CATEGORIES_FROM_NAMESPACE(perfetto)

// Register the set of available categories by passing a list of categories to
// this macro: perfetto::Category("cat1"), perfetto::Category("cat2"), ...
// `ns` is the name of the namespace in which the categories should be declared.
#define PERFETTO_DEFINE_CATEGORIES_IN_NAMESPACE(ns, ...) \
  PERFETTO_DEFINE_CATEGORIES_IN_NAMESPACE_WITH_ATTRS(    \
      ns, PERFETTO_COMPONENT_EXPORT, __VA_ARGS__)

// Register the set of available categories by passing a list of categories to
// this macro: perfetto::Category("cat1"), perfetto::Category("cat2"), ...
// `ns` is the name of the namespace in which the categories should be declared.
// `attrs` are linkage attributes for the underlying data source. See
// PERFETTO_DECLARE_DATA_SOURCE_STATIC_MEMBERS_WITH_ATTRS.
//
// Implementation note: the extra namespace (PERFETTO_TRACK_EVENT_NAMESPACE) is
// kept here only for backward compatibility.
#define PERFETTO_DEFINE_CATEGORIES_IN_NAMESPACE_WITH_ATTRS(ns, attrs, ...) \
  namespace ns {                                                           \
  namespace PERFETTO_TRACK_EVENT_NAMESPACE {                               \
  /* The list of category names */                                         \
  PERFETTO_INTERNAL_DECLARE_CATEGORIES(attrs, __VA_ARGS__)                 \
  /* The track event data source for this set of categories */             \
  PERFETTO_INTERNAL_DECLARE_TRACK_EVENT_DATA_SOURCE(attrs);                \
  } /* namespace PERFETTO_TRACK_EVENT_NAMESPACE  */                        \
  using PERFETTO_TRACK_EVENT_NAMESPACE::TrackEvent;                        \
  } /* namespace ns */                                                     \
  PERFETTO_DECLARE_DATA_SOURCE_STATIC_MEMBERS_WITH_ATTRS(                  \
      attrs, ns::PERFETTO_TRACK_EVENT_NAMESPACE::TrackEvent,               \
      ::perfetto::internal::TrackEventDataSourceTraits)

// Defines data structures for backing a category registry.
//
// Each category has one enabled/disabled bit per possible data source instance.
// The bits are packed, i.e., each byte holds the state for instances. To
// improve cache locality, the bits for each instance are stored separately from
// the names of the categories:
//
//   byte 0                      byte 1
//   (inst0, inst1, ..., inst7), (inst0, inst1, ..., inst7)
//
#define PERFETTO_INTERNAL_DECLARE_CATEGORIES(attrs, ...)                      \
  namespace internal {                                                        \
  constexpr ::perfetto::Category kCategories[] = {__VA_ARGS__};               \
  constexpr size_t kCategoryCount =                                           \
      sizeof(kCategories) / sizeof(kCategories[0]);                           \
  /* The per-instance enable/disable state per category */                    \
  attrs extern std::atomic<uint8_t> g_category_state_storage[kCategoryCount]; \
  /* The category registry which mediates access to the above structures. */  \
  /* The registry is used for two purposes: */                                \
  /**/                                                                        \
  /*    1) For looking up categories at build (constexpr) time. */            \
  /*    2) For declaring the per-namespace TrackEvent data source. */         \
  /**/                                                                        \
  /* Because usage #1 requires a constexpr type and usage #2 requires an */   \
  /* extern type (to avoid declaring a type based on a translation-unit */    \
  /* variable), we need two separate copies of the registry with different */ \
  /* storage specifiers. */                                                   \
  /**/                                                                        \
  /* Note that because of a Clang/Windows bug, the constexpr category */      \
  /* registry isn't given the enabled/disabled state array. All access */     \
  /* to the category states should therefore be done through the */           \
  /* non-constexpr registry. See */                                           \
  /* https://bugs.llvm.org/show_bug.cgi?id=51558 */                           \
  /**/                                                                        \
  /* TODO(skyostil): Unify these using a C++17 inline constexpr variable. */  \
  constexpr ::perfetto::internal::TrackEventCategoryRegistry                  \
      kConstExprCategoryRegistry(kCategoryCount, &kCategories[0], nullptr);   \
  attrs extern const ::perfetto::internal::TrackEventCategoryRegistry         \
      kCategoryRegistry;                                                      \
  static_assert(kConstExprCategoryRegistry.ValidateCategories(),              \
                "Invalid category names found");                              \
  }  // namespace internal
```
**  将上述"rendering"、"network.debug"、"audio.latency"进行声明，创建constexpr ::perfetto::internal::TrackEventCategoryRegistry类型
  的变量kConstExprCategoryRegistry，声明const ::perfetto::internal::TrackEventCategoryRegistry变量kCategoryRegistry**

```c++
  // Defines the TrackEvent data source for the current track event namespace.
// `virtual ~TrackEvent` is added to avoid `-Wweak-vtables` warning.
// Learn more : aosp/2019906
#define PERFETTO_INTERNAL_DECLARE_TRACK_EVENT_DATA_SOURCE(attrs)               \
  struct attrs TrackEvent : public ::perfetto::internal::TrackEventDataSource< \
                                TrackEvent, &internal::kCategoryRegistry> {    \
    virtual ~TrackEvent();                                                     \
  }
```
**声明使用的TrackEvent类型（命名空间为perfetto::perfetto_track_event）。**

```c++
// Similar to `PERFETTO_DECLARE_DATA_SOURCE_STATIC_MEMBERS` but it also takes
// custom attributes, which are useful when DataSource is defined in a component
// where a component specific export macro is used.
#define PERFETTO_DECLARE_DATA_SOURCE_STATIC_MEMBERS_WITH_ATTRS(attrs, ...) \
  template <>                                                              \
  attrs perfetto::internal::DataSourceType&                                \
  perfetto::DataSourceHelper<__VA_ARGS__>::type()
```
**类型函数声明**

**以上在trace_categories.cc中有类型定义：**
```c++
// Reserves internal static storage for our tracing categories.
PERFETTO_TRACK_EVENT_STATIC_STORAGE();

// Allocate storage for each category by using this macro once per track event
// namespace. `ns` is the name of the namespace in which the categories should
// be declared and `attrs` specify linkage attributes for the data source.
#define PERFETTO_TRACK_EVENT_STATIC_STORAGE_IN_NAMESPACE_WITH_ATTRS(ns, attrs) \
  namespace ns {                                                               \
  namespace PERFETTO_TRACK_EVENT_NAMESPACE {                                   \
  PERFETTO_INTERNAL_CATEGORY_STORAGE(attrs)                                    \
  PERFETTO_INTERNAL_DEFINE_TRACK_EVENT_DATA_SOURCE()                           \
  } /* namespace PERFETTO_TRACK_EVENT_NAMESPACE */                             \
  } /* namespace ns */                                                         \
  PERFETTO_DEFINE_DATA_SOURCE_STATIC_MEMBERS_WITH_ATTRS(                       \
      attrs, ns::PERFETTO_TRACK_EVENT_NAMESPACE::TrackEvent,                   \
      ::perfetto::internal::TrackEventDataSourceTraits)

// Allocate storage for each category by using this macro once per track event
// namespace.
#define PERFETTO_TRACK_EVENT_STATIC_STORAGE_IN_NAMESPACE(ns)   \
  PERFETTO_TRACK_EVENT_STATIC_STORAGE_IN_NAMESPACE_WITH_ATTRS( \
      ns, PERFETTO_COMPONENT_EXPORT)

// Allocate storage for each category by using this macro once per track event
// namespace.
#define PERFETTO_TRACK_EVENT_STATIC_STORAGE() \
  PERFETTO_TRACK_EVENT_STATIC_STORAGE_IN_NAMESPACE(perfetto)

// In a .cc file, declares storage for each category's runtime state.
#define PERFETTO_INTERNAL_CATEGORY_STORAGE(attrs)                      \
  namespace internal {                                                 \
  attrs std::atomic<uint8_t> g_category_state_storage[kCategoryCount]; \
  attrs const ::perfetto::internal::TrackEventCategoryRegistry         \
      kCategoryRegistry(kCategoryCount,                                \
                        &kCategories[0],                               \
                        &g_category_state_storage[0]);                 \
  }  // namespace internal

#define PERFETTO_INTERNAL_DEFINE_TRACK_EVENT_DATA_SOURCE() \
  TrackEvent::~TrackEvent() = default;

// Similar to `PERFETTO_DEFINE_DATA_SOURCE_STATIC_MEMBERS` but it also takes
// custom attributes, which are useful when DataSource is defined in a component
// where a component specific export macro is used.
#define PERFETTO_DEFINE_DATA_SOURCE_STATIC_MEMBERS_WITH_ATTRS(attrs, ...) \
  template <>                                                             \
  perfetto::internal::DataSourceType&                                     \
  perfetto::DataSourceHelper<__VA_ARGS__>::type() {                       \
    static perfetto::internal::DataSourceType type_;                      \
    return type_;                                                         \
  }                                                                       \
  PERFETTO_INTERNAL_SWALLOW_SEMICOLON()

// If placed at the end of a macro declaration, eats the semicolon at the end of
// the macro invocation (e.g., "MACRO(...);") to avoid warnings about extra
// semicolons.
#define PERFETTO_INTERNAL_SWALLOW_SEMICOLON() \
  extern int perfetto_internal_unused
```

**以上类型、类相关的声明、定义完成，大部分包含在宏定义中，使得理解较为复杂。**

# 二、void InitializePerfetto()
## 1、perfetto::Tracing::Initialize(args);
**static inline void Initialize(const TracingInitArgs& args) PERFETTO_ALWAYS_INLINE
其中“PERFETTO_ALWAYS_INLINE”表示类型安全检测，资源安全，要求加锁访问等
=》void Tracing::InitializeInternal(const TracingInitArgs& args)=》internal::TracingMuxerImpl::InitializeInstance(args);创建一个TracingMuxerImpl的单例，在构造函数中进行task_runner_的初始化（一些异步动作均可在里面完成，可以认为为任务单独创建的线程），后续在该异步线程中调用Initialize和AddBackends函数，Initilize进行fake赋值，AddBackends首先通过之前在perfetto::Tracing::Initialize中设置的in_process_backend_factory_函数指针构建internal::InProcessTracingBackend单例，改对象类继承TracingBackend（又继承TracingProducerBackend、TracingConsumerBackend）。
调用AddProducerBackend（主要将构建的RegisteredProducerBackend rb对象添加进producer_backends_的list，rb对象的producer变量指向并构建ProducerImpl类对象，感觉是效仿perfetto三大进程producer、service、consumer的执行体，调用InProcessSharedMemory创建共享内存；接着调用internal::InProcessTracingBackend类对象的ConnectProducer，模拟ipc producer和service连接，在此过程中调用GetOrCreateService函数创建TracingService对象，执行体TracingServiceImpl类对象，使用InProcessSharedMemory共享内存，紧接着调用TracingServiceImpl的ConnectProducer函数构建ProducerEndpointImpl对象（指令交互媒介），调用ProducerImpl的OnConnect函数连接TracingMuxerImpl中的datasource和TracingService中的datasource，SendOnConnectTriggers函数进行connect事件跟踪记录；调用ProducerImpl的Initialize函数进行ProducerEndpoint析构函数设置，一些变量赋值is_producer_provided_smb_ = endpoint->shared_memory();）。
调用AddConsumerBackend函数，构建的RegisteredProducerBackend rb对象添加进consumer_backends_的list，这里仅做简单的赋值。**