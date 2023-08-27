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
```c++
static inline void Initialize(const TracingInitArgs& args) PERFETTO_ALWAYS_INLINE  
```  

其中“PERFETTO_ALWAYS_INLINE”表示类型安全检测，资源安全，要求加锁访问等
=》void Tracing::InitializeInternal(const TracingInitArgs& args)=》internal::TracingMuxerImpl::InitializeInstance(args);创建一个TracingMuxerImpl的单例，在构造函数中进行task_runner_的初始化（一些异步动作均可在里面完成，可以认为为任务单独创建的线程），后续在该异步线程中调用Initialize和AddBackends函数，Initilize进行fake赋值，AddBackends首先通过之前在perfetto::Tracing::Initialize中设置的in_process_backend_factory_函数指针构建internal::InProcessTracingBackend单例，改对象类继承TracingBackend（又继承TracingProducerBackend、TracingConsumerBackend）。  

调用AddProducerBackend（主要将构建的RegisteredProducerBackend rb对象添加进producer_backends_的list，rb对象的producer变量指向并构建ProducerImpl类对象，感觉是效仿perfetto三大进程producer、service、consumer的执行体，调用InProcessSharedMemory创建共享内存；接着调用internal::InProcessTracingBackend类对象的ConnectProducer，模拟ipc producer和service连接，在此过程中调用GetOrCreateService函数创建TracingService对象，执行体TracingServiceImpl类对象，使用InProcessSharedMemory共享内存，紧接着调用TracingServiceImpl的ConnectProducer函数构建ProducerEndpointImpl对象（指令交互媒介），调用ProducerImpl的OnConnect函数连接TracingMuxerImpl中的datasource和TracingService中的datasource，SendOnConnectTriggers函数进行connect事件跟踪记录；调用ProducerImpl的Initialize函数进行ProducerEndpoint析构函数设置，一些变量赋值is_producer_provided_smb_ = endpoint->shared_memory();）。   

调用AddConsumerBackend函数，构建的RegisteredProducerBackend rb对象添加进consumer_backends_的list，这里仅做简单的赋值。
=》void TrackRegistry::InitializeInstance()=》instance_ = new TrackRegistry();    

## 2、perfetto::TrackEvent::Register();
上述提到perfetto::TrackEvent继承自::perfetto::internal::TrackEventDataSource，因此调用的是TrackEventDataSource中的静态函数Register()，=》
```c++
  // Initialize the track event library. Should be called before tracing is
  // enabled.
  static bool Register() {
    // Registration is performed out-of-line so users don't need to depend on
    // DataSourceDescriptor C++ bindings.
    return TrackEventInternal::Initialize(
        *Registry,
        [](const DataSourceDescriptor& dsd) { return Base::Register(dsd); });
  }
```
其中Registry即为&internal::kCategoryRegistry=》
```c++
attrs const ::perfetto::internal::TrackEventCategoryRegistry         \
      kCategoryRegistry(kCategoryCount,                                \
                        &kCategories[0],                               \
                        &g_category_state_storage[0]);
```
之后调用
```c++
bool TrackEventInternal::Initialize(
    const TrackEventCategoryRegistry& registry,
    bool (*register_data_source)(const DataSourceDescriptor&))
```
内部构建DataSourceDescriptor dsd，设置名为“track_event”，填充之前申请的category，返回调用register_data_source(dsd)，即调用基类DataSource的Register函数=》
```c++
  template <class... Args>
  static bool Register(const DataSourceDescriptor& descriptor,
                       const Args&... constructor_args) {
    // Silences -Wunused-variable warning in case the trace method is not used
    // by the translation unit that declares the data source.
    (void)tls_state_;

    auto factory = [constructor_args...]() {
      return std::unique_ptr<DataSourceBase>(
          new DerivedDataSource(constructor_args...));
    };
    internal::DataSourceParams params{
        DerivedDataSource::kSupportsMultipleInstances,
        DerivedDataSource::kRequiresCallbacksUnderLock};
    return Helper::type().Register(
        descriptor, factory, params, DerivedDataSource::kBufferExhaustedPolicy,
        GetCreateTlsFn(
            static_cast<typename DataSourceTraits::TlsStateType*>(nullptr)),
        GetCreateIncrementalStateFn(
            static_cast<typename DataSourceTraits::IncrementalStateType*>(
                nullptr)),
        nullptr);
  }
```
由于using Helper = DataSourceHelper<DerivedDataSource, DataSourceTraits>; 因此调用DataSourceHelper<TrackEvent, DefaultDataSourceTraits>的type()返回静态对象的Register函数
```c++
template <typename DerivedDataSource,
          typename DataSourceTraits = DefaultDataSourceTraits>
struct DataSourceHelper {
  static internal::DataSourceType& type() {
    static perfetto::internal::DataSourceType type_;
    return type_;
  }
};
```
```c++
  // Registers the data source type with the central tracing muxer.
  // * `descriptor` is the data source protobuf descriptor.
  // * `factory` is a std::function used to create instances of the data source
  //   type.
  // * `buffer_exhausted_policy` specifies what to do when the shared memory
  //   buffer runs out of chunks.
  // * `create_custom_tls_fn` and `create_incremental_state_fn` are function
  //   pointers called to create custom state. They will receive `user_arg` as
  //   an extra param.
  bool Register(const DataSourceDescriptor& descriptor,
                TracingMuxer::DataSourceFactory factory,
                internal::DataSourceParams params,
                BufferExhaustedPolicy buffer_exhausted_policy,
                CreateCustomTlsFn create_custom_tls_fn,
                CreateIncrementalStateFn create_incremental_state_fn,
                void* user_arg) {
    buffer_exhausted_policy_ = buffer_exhausted_policy;
    create_custom_tls_fn_ = create_custom_tls_fn;
    create_incremental_state_fn_ = create_incremental_state_fn;
    user_arg_ = user_arg;
    auto* tracing_impl = TracingMuxer::Get();
    return tracing_impl->RegisterDataSource(descriptor, factory, params,
                                            &state_);
  }
```
最后会调用TracingMuxerImpl类单例对象的RegisterDataSouce函数=》void TracingMuxerImpl::UpdateDataSourceOnAllBackends(RegisteredDataSource& rds, bool is_changed)=》void TracingServiceImpl::RegisterDataSource(ProducerID producer_id, const DataSourceDescriptor& desc)使得TracingServiceImpl也添加data_source_。   

# 三、std::unique_ptr<perfetto::TracingSession> StartTracing()
```c++
std::unique_ptr<perfetto::TracingSession> StartTracing() {
  // The trace config defines which types of data sources are enabled for
  // recording. In this example we just need the "track_event" data source,
  // which corresponds to the TRACE_EVENT trace points.
  perfetto::TraceConfig cfg;
  cfg.add_buffers()->set_size_kb(1024);
  auto* ds_cfg = cfg.add_data_sources()->mutable_config();
  ds_cfg->set_name("track_event");

  auto tracing_session = perfetto::Tracing::NewTrace();
  tracing_session->Setup(cfg);
  tracing_session->StartBlocking();
  return tracing_session;
}
```
## 1、perfetto::Tracing::NewTrace()
=》std::unique_ptr<TracingSession> Tracing::NewTraceInternal(BackendType backend, TracingConsumerBackend(*system_backend_factory)())
=》std::unique_ptr<TracingSession> TracingMuxerImpl::CreateTracingSession(BackendType requested_backend_type, TracingConsumerBackend* (*system_backend_factory)())
在consumer_backends_的list中添加ConsumerImpl类
```c++
  struct RegisteredProducerBackend {
    // Backends are supposed to have static lifetime.
    TracingProducerBackend* backend = nullptr;
    TracingBackendId id = 0;
    BackendType type{};

    TracingBackend::ConnectProducerArgs producer_conn_args;
    std::unique_ptr<ProducerImpl> producer;

    std::vector<RegisteredStartupSession> startup_sessions;
  };

  struct RegisteredConsumerBackend {
    // Backends are supposed to have static lifetime.
    TracingConsumerBackend* backend = nullptr;
    BackendType type{};
    // The calling code can request more than one concurrently active tracing
    // session for the same backend. We need to create one consumer per session.
    std::vector<std::unique_ptr<ConsumerImpl>> consumers;
  };
```
由于policy_==nullptr，进行
```c++
if (!policy_) {
  InitializeConsumer(session_id);
  return;
}
```
```c++
void TracingMuxerImpl::InitializeConsumer(TracingSessionGlobalID session_id) {
  PERFETTO_DCHECK_THREAD(thread_checker_);

  auto res = FindConsumerAndBackend(session_id);
  if (!res.first || !res.second)
    return;
  TracingMuxerImpl::ConsumerImpl* consumer = res.first;
  RegisteredConsumerBackend& backend = *res.second;

  TracingBackend::ConnectConsumerArgs conn_args;
  conn_args.consumer = consumer;
  conn_args.task_runner = task_runner_.get();
  consumer->Initialize(backend.backend->ConnectConsumer(conn_args));
}
```
操作和producer类似，ConnectConsumer中调用TracingServiceImpl的ConnectConsumer函数创建ConsumerEndpointImpl类对象，之后调用consumer的OnConnect函数以及Initialize函数。   

## 2、tracing_session->Setup(cfg);
=》```c++
void TracingMuxerImpl::TracingSessionImpl::Setup(const TraceConfig& cfg,
                                                 int fd)
```
=》```c++
void TracingMuxerImpl::SetupTracingSession(
    TracingSessionGlobalID session_id,
    const std::shared_ptr<TraceConfig>& trace_config,
    base::ScopedFile trace_fd) {
  PERFETTO_DCHECK_THREAD(thread_checker_);
  PERFETTO_CHECK(!trace_fd || trace_config->write_into_file());

  auto* consumer = FindConsumer(session_id);
  if (!consumer)
    return;

  consumer->trace_config_ = trace_config;
  if (trace_fd)
    consumer->trace_fd_ = std::move(trace_fd);

  if (!consumer->connected_)
    return;

  // Only used in the deferred start mode.
  if (trace_config->deferred_start()) {
    consumer->service_->EnableTracing(*trace_config,
                                      std::move(consumer->trace_fd_));
  }
}
```

## 3、tracing_session->StartBlocking();
=》```c++
// Can be called from any thread except the service thread.
void TracingMuxerImpl::TracingSessionImpl::StartBlocking() {
  PERFETTO_DCHECK(!muxer_->task_runner_->RunsTasksOnCurrentThread());
  auto* muxer = muxer_;
  auto session_id = session_id_;
  base::WaitableEvent tracing_started;
  muxer->task_runner_->PostTask([muxer, session_id, &tracing_started] {
    auto* consumer = muxer->FindConsumer(session_id);
    if (!consumer) {
      // TODO(skyostil): Signal an error to the user.
      tracing_started.Notify();
      return;
    }
    PERFETTO_DCHECK(!consumer->blocking_start_complete_callback_);
    consumer->blocking_start_complete_callback_ = [&] {
      tracing_started.Notify();
    };
    muxer->StartTracingSession(session_id);
  });
  tracing_started.Wait();
}
```
=》```c++
void TracingMuxerImpl::StartTracingSession(TracingSessionGlobalID session_id)
```
=》```c++
base::Status TracingServiceImpl::EnableTracing(ConsumerEndpointImpl* consumer,
                                               const TraceConfig& cfg,
                                               base::ScopedFile fd)
```
=》```c++
TracingServiceImpl::DataSourceInstance* TracingServiceImpl::SetupDataSource(
    const TraceConfig::DataSource& cfg_data_source,
    const TraceConfig::ProducerConfig& producer_config,
    const RegisteredDataSource& data_source,
    TracingSession* tracing_session)
```
=》```c++
producer->SetupSharedMemory(std::move(shared_memory), page_size,
                                /*provided_by_producer=*/false);
producer->SetupDataSource(inst_id, ds_config);
```
=》```c++
void TracingMuxerImpl::SetupDataSource(TracingBackendId backend_id,
                                       uint32_t backend_connection_id,
                                       DataSourceInstanceID instance_id,
                                       const DataSourceConfig& cfg) 
```
=》```c++
SetupDataSourceImpl(rds, backend_id, backend_connection_id, instance_id,
                        cfg, /*startup_session_id=*/0);
```
=》```c++
internal_state->data_source->OnSetup(setup_args);
```
=》```c++
TrackEventInternal::EnableTracing(*Registry, config_, args);
```
=》```c++
// static
void TrackEventInternal::EnableTracing(
    const TrackEventCategoryRegistry& registry,
    const protos::gen::TrackEventConfig& config,
    const DataSourceBase::SetupArgs& args) {
  for (size_t i = 0; i < registry.category_count(); i++) {
    if (IsCategoryEnabled(registry, config, *registry.GetCategory(i)))
      registry.EnableCategoryForInstance(i, args.internal_instance_index);
  }
  TrackEventSessionObserverRegistry::GetInstance()->ForEachObserverForRegistry(
      registry, [&](TrackEventSessionObserver* o) { o->OnSetup(args); });
}
```
=》```c++
void TrackEventCategoryRegistry::EnableCategoryForInstance(
    size_t category_index,
    uint32_t instance_index) const {
  PERFETTO_DCHECK(instance_index < kMaxDataSourceInstances);
  PERFETTO_DCHECK(category_index < category_count_);
  // Matches the acquire_load in DataSource::Trace().
  state_storage_[category_index].fetch_or(
      static_cast<uint8_t>(1u << instance_index), std::memory_order_release);
}
```

# 四、TRACE_COUNTER("rendering", "Framerate", 120);
=》```c++
#define TRACE_COUNTER(category, track, ...)                 \
  PERFETTO_INTERNAL_TRACK_EVENT_WITH_METHOD(                \
      TraceForCategory, category, /*name=*/nullptr,         \
      ::perfetto::protos::pbzero::TrackEvent::TYPE_COUNTER, \
      ::perfetto::CounterTrack(track), ##__VA_ARGS__)

// Efficiently determines whether tracing is enabled for the given category, and
// if so, emits one trace event with the given arguments.
#define PERFETTO_INTERNAL_TRACK_EVENT_WITH_METHOD(method, category, name, ...) \
  do {                                                                         \
    ::perfetto::internal::ValidateEventNameType<decltype(name)>();             \
    namespace tns = PERFETTO_TRACK_EVENT_NAMESPACE;                            \
    /* Compute the category index outside the lambda to work around a */       \
    /* GCC 7 bug */                                                            \
    constexpr auto PERFETTO_UID(                                               \
        kCatIndex_ADD_TO_PERFETTO_DEFINE_CATEGORIES_IF_FAILS_) =               \
        PERFETTO_GET_CATEGORY_INDEX(category);                                 \
    if (::PERFETTO_TRACK_EVENT_NAMESPACE::internal::IsDynamicCategory(         \
            category)) {                                                       \
      tns::TrackEvent::CallIfEnabled(                                          \
          [&](uint32_t instances) PERFETTO_NO_THREAD_SAFETY_ANALYSIS {         \
            tns::TrackEvent::method(instances, category, name, ##__VA_ARGS__); \
          });                                                                  \
    } else {                                                                   \
      tns::TrackEvent::CallIfCategoryEnabled(                                  \
          PERFETTO_UID(kCatIndex_ADD_TO_PERFETTO_DEFINE_CATEGORIES_IF_FAILS_), \
          [&](uint32_t instances) PERFETTO_NO_THREAD_SAFETY_ANALYSIS {         \
            tns::TrackEvent::method(                                           \
                instances,                                                     \
                PERFETTO_UID(                                                  \
                    kCatIndex_ADD_TO_PERFETTO_DEFINE_CATEGORIES_IF_FAILS_),    \
                name, ##__VA_ARGS__);                                          \
          });                                                                  \
    }                                                                          \
  } while (false)
```
=》```c++
  // The following methods forward all arguments to TraceForCategoryBody
  // while casting string constants to const char* and integer arguments to
  // int64_t, uint64_t or bool.
  template <typename CategoryType,
            typename EventNameType,
            typename... Arguments>
  static void TraceForCategory(uint32_t instances,
                               const CategoryType& category,
                               const EventNameType& name,
                               perfetto::protos::pbzero::TrackEvent::Type type,
                               Arguments&&... args) PERFETTO_ALWAYS_INLINE {
    TraceForCategoryBody(instances, DecayStrType(category), DecayStrType(name),
                         type, DecayArgType(args)...);
  }
```
=》```c++
// Trace point with with a timestamp and a counter sample.
  template <typename CategoryType,
            typename EventNameType,
            typename TimestampType = uint64_t,
            typename TimestampTypeCheck = typename std::enable_if<
                IsValidTimestamp<TimestampType>()>::type,
            typename ValueType>
  static void TraceForCategoryBody(
      uint32_t instances,
      const CategoryType& category,
      const EventNameType&,
      perfetto::protos::pbzero::TrackEvent::Type type,
      CounterTrack track,
      TimestampType timestamp,
      ValueType value) PERFETTO_NO_INLINE {
    PERFETTO_DCHECK(type == perfetto::protos::pbzero::TrackEvent::TYPE_COUNTER);
    TraceForCategoryImpl(
        instances, category, /*name=*/nullptr, type, track, timestamp,
        [&](EventContext event_ctx) {
          if (std::is_integral<ValueType>::value) {
            int64_t value_int64 = static_cast<int64_t>(value);
            if (track.is_incremental()) {
              TrackEventIncrementalState* incr_state =
                  event_ctx.GetIncrementalState();
              PERFETTO_DCHECK(incr_state != nullptr);
              auto prv_value =
                  incr_state->last_counter_value_per_track[track.uuid];
              event_ctx.event()->set_counter_value(value_int64 - prv_value);
              prv_value = value_int64;
              incr_state->last_counter_value_per_track[track.uuid] = prv_value;
            } else {
              event_ctx.event()->set_counter_value(value_int64);
            }
          } else {
            event_ctx.event()->set_double_counter_value(
                static_cast<double>(value));
          }
        });
  }
```
=》```c++
template <typename CategoryType,
            typename EventNameType,
            typename TrackType = Track,
            typename TimestampType = uint64_t,
            typename TimestampTypeCheck = typename std::enable_if<
                IsValidTimestamp<TimestampType>()>::type,
            typename TrackTypeCheck =
                typename std::enable_if<IsValidTrack<TrackType>()>::type,
            typename... Arguments>
  static void TraceForCategoryImpl(
      uint32_t instances,
      const CategoryType& category,
      const EventNameType& event_name,
      perfetto::protos::pbzero::TrackEvent::Type type,
      const TrackType& track,
      const TimestampType& timestamp,
      Arguments&&... args) PERFETTO_ALWAYS_INLINE {
    using CatTraits = CategoryTraits<CategoryType>;
    TraceWithInstances(
        instances, category, [&](typename Base::TraceContext ctx) {
          // If this category is dynamic, first check whether it's enabled.
          if (CatTraits::kIsDynamic &&
              !IsDynamicCategoryEnabled(
                  &ctx, CatTraits::GetDynamicCategory(category))) {
            return;
          }

          auto event_ctx = WriteTrackEvent(ctx, category, event_name, type,
                                           track, timestamp);
          WriteTrackEventArgs(std::move(event_ctx),
                              std::forward<Arguments>(args)...);
        });
  }
```
=》```c++
template <typename Traits = DefaultTracePointTraits, typename Lambda>
  static void TraceWithInstances(
      uint32_t cached_instances,
      Lambda tracing_fn,
      typename Traits::TracePointData trace_point_data = {}) {
    PERFETTO_DCHECK(cached_instances);

    if (!Helper::type().template TracePrologue<DataSourceTraits, Traits>(
            &tls_state_, &cached_instances, trace_point_data)) {
      return;
    }

    for (internal::DataSourceType::InstancesIterator it =
             Helper::type().template BeginIteration<Traits>(
                 cached_instances, tls_state_, trace_point_data);
         it.instance; Helper::type().template NextIteration<Traits>(
             &it, tls_state_, trace_point_data)) {
      tracing_fn(TraceContext(it.instance, it.i));
    }

    Helper::type().TraceEpilogue(tls_state_);
  }
```
其中TracePrologue函数构建*tls_state = GetOrCreateDataSourceTLS<DataSourceTraits>();此后会找到之前注册对应的DataSourceType的instance
```c++
struct InstancesIterator {
    // A bitmap of the currenly active instances.
    uint32_t cached_instances;
    // The current instance index.
    uint32_t i;
    // The current instance. If this is `nullptr`, the iteration is over.
    DataSourceInstanceThreadLocalState* instance;
  };

  // Returns an iterator to the active instances of this data source type.
  //
  // `cached_instances` is a copy of the bitmap of the active instances for this
  // data source type (usually just a copy of ValidInstances(), but can be
  // customized).
  //
  // `tls_state` is the thread local pointer obtained from TracePrologue.
  //
  // `TracePointTraits` and `trace_point_data` are customization point for
  // getting the active instances bitmap.
  template <typename TracePointTraits>
  InstancesIterator BeginIteration(
      uint32_t cached_instances,
      DataSourceThreadLocalState* tls_state,
      typename TracePointTraits::TracePointData trace_point_data) {
    InstancesIterator it{};
    it.cached_instances = cached_instances;
    FirstActiveInstance<TracePointTraits>(&it, tls_state, trace_point_data);
    return it;
  }
```
接下来就是添加对象进行value值记录，中间涉及较多的函数指针，理解不易。