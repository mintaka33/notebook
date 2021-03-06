i915 command buffer submission flow
====

# i915_request_queue

```c
// i915_gem_execbuffer.c
i915_gem_do_execbuffer(struct drm_device *dev,
		       struct drm_file *file,
		       struct drm_i915_gem_execbuffer2 *args,
		       struct drm_i915_gem_exec_object2 *exec,
		       struct drm_syncobj **fences)
{
    trace_i915_request_queue(eb.request, eb.batch_flags);
}
```

# i915_request_add

```c
// i915_gem_execbuffer.c
i915_gem_do_execbuffer(struct drm_device *dev,
		       struct drm_file *file,
		       struct drm_i915_gem_execbuffer2 *args,
		       struct drm_i915_gem_exec_object2 *exec,
		       struct drm_syncobj **fences)
{
    __i915_request_add(eb.request, err == 0);
}

// i915_request.c
void __i915_request_add(struct i915_request *request, bool flush_caches)
{trace_i915_request_add(request);}
```

# i915_request_submit

**sw_fence_commit**

```c
void __i915_request_add(struct i915_request *request, bool flush_caches)
{
	/*
	 * Let the backend know a new request has arrived that may need
	 * to adjust the existing execution schedule due to a high priority
	 * request - i.e. we may want to preempt the current request in order
	 * to run a high priority dependency chain *before* we can execute this
	 * request.
	 *
	 * This is called before the request is ready to run so that we can
	 * decide whether to preempt the entire chain so that it is ready to
	 * run at the earliest possible convenience.
	 */
	local_bh_disable();
	rcu_read_lock(); /* RCU serialisation for set-wedged protection */
	if (engine->schedule)
		engine->schedule(request, &request->ctx->sched);
	rcu_read_unlock();
	i915_sw_fence_commit(&request->submit);
	local_bh_enable(); /* Kick the execlists tasklet if just scheduled */
}

void i915_sw_fence_commit(struct i915_sw_fence *fence)
{
	debug_fence_activate(fence);
	i915_sw_fence_complete(fence);
}
```

**engine schedule**

```c
static void execlists_set_default_submission(struct intel_engine_cs *engine)
{
	engine->submit_request = execlists_submit_request;
	engine->cancel_requests = execlists_cancel_requests;
	engine->schedule = execlists_schedule;
	engine->execlists.tasklet.func = execlists_submission_tasklet;
}
```

**execlists_schedule**

```c
static void execlists_schedule(struct i915_request *request,
			       const struct i915_sched_attr *attr)
{
	struct i915_priolist *uninitialized_var(pl);
	struct intel_engine_cs *engine, *last;
	struct i915_dependency *dep, *p;
	struct i915_dependency stack;
	const int prio = attr->priority;
	LIST_HEAD(dfs);

	GEM_BUG_ON(prio == I915_PRIORITY_INVALID);

	if (i915_request_completed(request))
		return;

	if (prio <= READ_ONCE(request->sched.attr.priority))
		return;

	/* Need BKL in order to use the temporary link inside i915_dependency */
	lockdep_assert_held(&request->i915->drm.struct_mutex);

	stack.signaler = &request->sched;
	list_add(&stack.dfs_link, &dfs);

	/*
	 * Recursively bump all dependent priorities to match the new request.
	 *
	 * A naive approach would be to use recursion:
	 * static void update_priorities(struct i915_sched_node *node, prio) {
	 *	list_for_each_entry(dep, &node->signalers_list, signal_link)
	 *		update_priorities(dep->signal, prio)
	 *	queue_request(node);
	 * }
	 * but that may have unlimited recursion depth and so runs a very
	 * real risk of overunning the kernel stack. Instead, we build
	 * a flat list of all dependencies starting with the current request.
	 * As we walk the list of dependencies, we add all of its dependencies
	 * to the end of the list (this may include an already visited
	 * request) and continue to walk onwards onto the new dependencies. The
	 * end result is a topological list of requests in reverse order, the
	 * last element in the list is the request we must execute first.
	 */
	list_for_each_entry(dep, &dfs, dfs_link) {
		struct i915_sched_node *node = dep->signaler;

		/*
		 * Within an engine, there can be no cycle, but we may
		 * refer to the same dependency chain multiple times
		 * (redundant dependencies are not eliminated) and across
		 * engines.
		 */
		list_for_each_entry(p, &node->signalers_list, signal_link) {
			GEM_BUG_ON(p == dep); /* no cycles! */

			if (i915_sched_node_signaled(p->signaler))
				continue;

			GEM_BUG_ON(p->signaler->attr.priority < node->attr.priority);
			if (prio > READ_ONCE(p->signaler->attr.priority))
				list_move_tail(&p->dfs_link, &dfs);
		}
	}

	/*
	 * If we didn't need to bump any existing priorities, and we haven't
	 * yet submitted this request (i.e. there is no potential race with
	 * execlists_submit_request()), we can set our own priority and skip
	 * acquiring the engine locks.
	 */
	if (request->sched.attr.priority == I915_PRIORITY_INVALID) {
		GEM_BUG_ON(!list_empty(&request->sched.link));
		request->sched.attr = *attr;
		if (stack.dfs_link.next == stack.dfs_link.prev)
			return;
		__list_del_entry(&stack.dfs_link);
	}

	last = NULL;
	engine = request->engine;
	spin_lock_irq(&engine->timeline.lock);

	/* Fifo and depth-first replacement ensure our deps execute before us */
	list_for_each_entry_safe_reverse(dep, p, &dfs, dfs_link) {
		struct i915_sched_node *node = dep->signaler;

		INIT_LIST_HEAD(&dep->dfs_link);

		engine = sched_lock_engine(node, engine);

		if (prio <= node->attr.priority)
			continue;

		node->attr.priority = prio;
		if (!list_empty(&node->link)) {
			if (last != engine) {
				pl = lookup_priolist(engine, prio);
				last = engine;
			}
			GEM_BUG_ON(pl->priority != prio);
			list_move_tail(&node->link, &pl->requests);
		}

		if (prio > engine->execlists.queue_priority &&
		    i915_sw_fence_done(&sched_to_request(node)->submit))
			__submit_queue(engine, prio);
	}

	spin_unlock_irq(&engine->timeline.lock);
}
```

**sw_fence_complete**

```c
static void i915_sw_fence_complete(struct i915_sw_fence *fence)
{
	debug_fence_assert(fence);

	if (WARN_ON(i915_sw_fence_done(fence)))
		return;

	__i915_sw_fence_complete(fence, NULL);
}
```

or

```c
static int i915_sw_fence_wake(wait_queue_entry_t *wq, unsigned mode, int flags, void *key)
{
	list_del(&wq->entry);
	__i915_sw_fence_complete(wq->private, key);

	if (wq->flags & I915_SW_FENCE_FLAG_ALLOC)
		kfree(wq);
	return 0;
}
```

**call submit_notify**

```c
static void __i915_sw_fence_complete(struct i915_sw_fence *fence,
				     struct list_head *continuation)
{
	debug_fence_assert(fence);

	if (!atomic_dec_and_test(&fence->pending))
		return;

	debug_fence_set_state(fence, DEBUG_FENCE_IDLE, DEBUG_FENCE_NOTIFY);

	if (__i915_sw_fence_notify(fence, FENCE_COMPLETE) != NOTIFY_DONE)
		return;

	debug_fence_set_state(fence, DEBUG_FENCE_NOTIFY, DEBUG_FENCE_IDLE);

	__i915_sw_fence_wake_up_all(fence, continuation);

	debug_fence_destroy(fence);
	__i915_sw_fence_notify(fence, FENCE_FREE);
}

static int __i915_sw_fence_notify(struct i915_sw_fence *fence,
				  enum i915_sw_fence_notify state)
{
	i915_sw_fence_notify_t fn;

	fn = (i915_sw_fence_notify_t)(fence->flags & I915_SW_FENCE_MASK);
	return fn(fence, state);
}
```

**submit_notify**

```c
submit_notify(struct i915_sw_fence *fence, enum i915_sw_fence_notify state)
{
	struct i915_request *request =
		container_of(fence, typeof(*request), submit);

	switch (state) {
	case FENCE_COMPLETE:
		trace_i915_request_submit(request);
		rcu_read_lock();
		request->engine->submit_request(request);
		rcu_read_unlock();
		break;

	case FENCE_FREE:
		i915_request_put(request);
		break;
	}

	return NOTIFY_DONE;
}

i915_request_alloc(struct intel_engine_cs *engine, struct i915_gem_context *ctx)
{i915_sw_fence_init(&i915_request_get(rq)->submit, submit_notify);}

i915_gem_do_execbuffer(struct drm_device *dev,
		       struct drm_file *file,
		       struct drm_i915_gem_execbuffer2 *args,
		       struct drm_i915_gem_exec_object2 *exec,
		       struct drm_syncobj **fences)
{
	/* Allocate a request for this batch buffer nice and early. */
	eb.request = i915_request_alloc(eb.engine, eb.ctx);
}
```

**submit_request**

```c
static void execlists_set_default_submission(struct intel_engine_cs *engine)
{
	engine->submit_request = execlists_submit_request;
	engine->cancel_requests = execlists_cancel_requests;
	engine->schedule = execlists_schedule;
	engine->execlists.tasklet.func = execlists_submission_tasklet;
}

static void execlists_submit_request(struct i915_request *request)
{
	queue_request(engine, &request->sched, rq_prio(request));
	submit_queue(engine, rq_prio(request));
}

static void submit_queue(struct intel_engine_cs *engine, int prio)
{
	if (prio > engine->execlists.queue_priority)
		__submit_queue(engine, prio);
}

static void __submit_queue(struct intel_engine_cs *engine, int prio)
{
	engine->execlists.queue_priority = prio;
	tasklet_hi_schedule(&engine->execlists.tasklet);
}

static inline void tasklet_hi_schedule(struct tasklet_struct *t)
{
	if (!test_and_set_bit(TASKLET_STATE_SCHED, &t->state))
		__tasklet_hi_schedule(t);
}

void __tasklet_hi_schedule(struct tasklet_struct *t)
{
	__tasklet_schedule_common(t, &tasklet_hi_vec,
				  HI_SOFTIRQ);
}

static void __tasklet_schedule_common(struct tasklet_struct *t,
				      struct tasklet_head __percpu *headp,
				      unsigned int softirq_nr)
{
	struct tasklet_head *head;
	unsigned long flags;

	local_irq_save(flags);
	head = this_cpu_ptr(headp);
	t->next = NULL;
	*head->tail = t;
	head->tail = &(t->next);
	raise_softirq_irqoff(softirq_nr);
	local_irq_restore(flags);
}
```

# i915_request_execute

**init platform**

```c
// i915_pci.c
#define GEN8_FEATURES \
	G75_FEATURES, \
	GEN(8), \
	BDW_COLORS, \
	.page_sizes = I915_GTT_PAGE_SIZE_4K | \
		      I915_GTT_PAGE_SIZE_2M, \
	.has_logical_ring_contexts = 1, \ // use lrc in Gen8+
	.has_full_48bit_ppgtt = 1, \
	.has_64bit_reloc = 1, \
	.has_reset_engine = 1
  
#define GEN9_FEATURES \
	GEN8_FEATURES, \
	GEN(9), \
	GEN9_DEFAULT_PAGE_SIZES, \
	.has_logical_ring_preemption = 1, \
	.has_csr = 1, \
	.has_guc = 1, \
	.has_ipc = 1, \
	.ddb_size = 896
  
#define CFL_PLATFORM \
	GEN9_FEATURES, \
	PLATFORM(INTEL_COFFEELAKE)

#define HAS_EXECLISTS(dev_priv) HAS_LOGICAL_RING_CONTEXTS(dev_priv)

#define HAS_LOGICAL_RING_CONTEXTS(dev_priv) \
		((dev_priv)->info.has_logical_ring_contexts)
```

**init func pointer**

```c
// intel_engine_cs.c
int intel_engines_init(struct drm_i915_private *dev_priv)
{
    if (HAS_EXECLISTS(dev_priv))
        init = class_info->init_execlists;
    else
       init = class_info->init_legacy;
}

// intel_engine_cs.c
static const struct engine_class_info intel_engine_classes[] = {
	[RENDER_CLASS] = {
		.name = "rcs",
		.init_execlists = logical_render_ring_init,
		.init_legacy = intel_init_render_ring_buffer,
		.uabi_class = I915_ENGINE_CLASS_RENDER,
	},
}

int logical_render_ring_init(struct intel_engine_cs *engine)
{logical_ring_setup(engine);}

logical_ring_setup(struct intel_engine_cs *engine)
{logical_ring_default_vfuncs(engine);}

logical_ring_default_vfuncs(struct intel_engine_cs *engine)
{engine->set_default_submission = execlists_set_default_submission;}

static void execlists_set_default_submission(struct intel_engine_cs *engine)
{
    engine->submit_request = execlists_submit_request;
    engine->cancel_requests = execlists_cancel_requests;
    engine->schedule = execlists_schedule;
    engine->execlists.tasklet.func = execlists_submission_tasklet;
}
```

**register**

```c
logical_ring_setup(struct intel_engine_cs *engine)
{
	tasklet_init(&engine->execlists.tasklet,
		     execlists_submission_tasklet, (unsigned long)engine);
}
```
or
```c
static void execlists_set_default_submission(struct intel_engine_cs *engine)
{engine->execlists.tasklet.func = execlists_submission_tasklet;}
```

**request_execute call flow**

```c
static void execlists_submission_tasklet(unsigned long data)
{execlists_dequeue(engine);}

static void execlists_dequeue(struct intel_engine_cs *engine)
{submit = __execlists_dequeue(engine);}

static bool __execlists_dequeue(struct intel_engine_cs *engine)
{__i915_request_submit(rq);}

void __i915_request_submit(struct i915_request *request)
{trace_i915_request_execute(request);}
```

# i915_request_in

```c
// intel_lrc.c
void __i915_request_submit(struct i915_request *request)
{trace_i915_request_in(rq, port_index(port, execlists));}
```
# i915_request_out

```c
static void execlists_submission_tasklet(unsigned long data)
{
    execlists_context_schedule_out(rq, INTEL_CONTEXT_SCHEDULE_OUT);
}

execlists_context_schedule_out(struct i915_request *rq, unsigned long status)
{
    intel_engine_context_out(rq->engine);	
    execlists_context_status_change(rq, status);
    trace_i915_request_out(rq);
}
```
