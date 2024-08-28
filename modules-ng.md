# Modules the Next Generation

To support the In-Service Software Update effort, we need to make reliably and
quickly bringup and teardown core primitives of FunOS -- that is, they are
attributes of the system which are functional guarantees and can serve as a
basis for other architectural features (most proximately and prominently, ISSU).

This is not to say that FunOS does not reliably boot and quiesce, but there is
currently no technological basis that guarantees the performance and
architectural behaviors of these paths, specifically with regard to dependency
handling.

Today's module system is an important step on this road and has demonstrated
that FunOS's functional units can be expressed via a statically-defined C
structure, which preserves the locality of reference between the module's code
and its metadata. But there are some areas which need to be improved upon before
we can be confident in modules as the basis for ISSU.

## Too many entry points
Modules today define several entry points that could be coalesced to reduce the
number of possible codepaths they can take. Module entry points today are:

- `.direct_init`
- `.json_update_callback'
- `.issu_drain`
- `.issu_quiesce`
- `.issu_save`
- `.unload`
- `.terminate`

In order to achieve a high reliability threshold for ISSU, we want to minimize
and/or eliminate ISSU-specific code paths and instead make ISSU a combination of
well-trodden paths and operations (i.e. firmware flash, boot, and shutdown). In
other words, we do not want modules to understand that an ISSU has happened.
Architecturally, making modules insensitive to this operation prevents
unintended dependencies from creeping in, but it also ensures that modules only
ever exercise one code path for initialization, configuration updates, state
restoration, and teardown. So we can introduce a new module API with the
following entry points:

### Initialize
The initializer entry points is called very early in boot for all modules. This
entry point is invoked *before* a module's dependencies have been enumerated or
started. Modules can use it to perform extremely basic operations, such as
zero-initializing memory, obtaining entropy, etc.

#### Example
```
struct security_state {
	uint32_t sec_entropy;
	uint32_t sec_debug;
	char sec_chip[32];
};

struct security_state __sec = {
	.sec_entropy = 0,
	.sec_debug = 0,
	.sec_chip = NULL,
	.sec_chip_mem = "",
};

static void
_security_module_init(const struct module *m)
{
	struct security_state *s = &__sec;
	s->sec_entropy = arc4random();
}
```

### Configuration Translate
This entry point is invoked when a new configuration for the module is received
by the system. Modules use this entry point to parse an input JSON blob and fill
in a configuration structure, which is then returned to the system. This is
a separate action from applying the configuration. By separating parsing and
ingestion, we open the door to supporting a different format for serialized
configuration, which will very likely be required by the confidential compute
variant of FunOS, where complex parsing logic is not desireable. Supporting such
a format would be a simple matter of adding a new parsing entry point for it.

#### Example
```
static const void *
_security_module_parse(
		const struct module *m,
		const struct fun_json *js,
		void *buff,
		size_t buff_len)
{
	bool yn = false;
	struct security_state *state = buff;

	if (buff_len != sizeof(struct security_state)) {
		panic("invalid configuration buffer");
	}

	// No big deal if this isn't in the blob.
	yn = fun_json_lookup_uint32(js, "debug", &state->sec_debug);
	if (!yn) {
		// Indicate that something was wrong with the JSON configuration, and it
		// was not accepted.
		return NULL;
	}

	return state;
}
```

### Configuration Update
This entry point is called when a new configuration update is received and has
been parsed successfully. The structure given to this entry point is a bit-for-
bit copy of the one returned from the configuration parsing entry point. The
caller is expected to update their internal state accordingly.

#### Example
```
static void
_security_module_update(
		const struct module *m,
		const void *buff,
		size_t buff_len)
{
	struct security_state *sec = &__sec;
	const struct security_state *in = buff;

	if (buff_len != sizeof(struct security_state)) {
		panic("invalid configuration buffer");
	}

	sec->sec_debug = in->sec_debug;
}
```

### Boot
The boot entry point is called when the module's dependencies have been
satisfied. In this entry point, the module can begin its actual operations.

## Module configuration namespace is shared

## JSON (probably) not suitable for Confidential FunOS

## Modules care too much about other modules

## Mutable and immutable state are intertwined
