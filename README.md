## Setup
These steps were tested on Ubuntu 22.04.

1) Install dependencies:
```bash
npm install
sudo apt install imagemagick
```

2) Make a wrangler.toml based on wrangler.example.toml by copying it and customising your account_id.

### BUG: workers-sites wrangler publish JavaScript heap out of memory #2223
This bug is a deal breaker, because it means that you can't upload larger sites from environments such as Github Actions
which don't have a huge amount of memory.

Download public.zip which contains a Next.js statically rendered site with 20,000 pages:

Extract public.zip into this project.

Make sure you **haven't** set the NODE_OPTIONS environment variable. Then run this command:
```bash
wrangler publish
```

You should get this error:
```bash
<--- Last few GCs --->

[606709:0x5981ee0]    21245 ms: Scavenge 4042.9 (4125.8) -> 4042.8 (4129.8) MB, 4.0 / 0.0 ms  (average mu = 0.765, current mu = 0.734) allocation failure 
[606709:0x5981ee0]    21260 ms: Scavenge 4046.7 (4133.5) -> 4046.4 (4133.7) MB, 4.0 / 0.0 ms  (average mu = 0.765, current mu = 0.734) allocation failure 
[606709:0x5981ee0]    21296 ms: Mark-sweep 4050.3 (4137.4) -> 4049.9 (4145.2) MB, 24.8 / 0.0 ms  (average mu = 0.727, current mu = 0.676) allocation failure scavenge might not succeed


<--- JS stacktrace --->

FATAL ERROR: Reached heap limit Allocation failed - JavaScript heap out of memory
 1: 0xb0a860 node::Abort() [/home/user/.nvm/versions/node/v16.16.0/bin/node]
 2: 0xa1c193 node::FatalError(char const*, char const*) [/home/user/.nvm/versions/node/v16.16.0/bin/node]
 3: 0xcf9a6e v8::Utils::ReportOOMFailure(v8::internal::Isolate*, char const*, bool) [/home/user/.nvm/versions/node/v16.16.0/bin/node]
 4: 0xcf9de7 v8::internal::V8::FatalProcessOutOfMemory(v8::internal::Isolate*, char const*, bool) [/home/user/.nvm/versions/node/v16.16.0/bin/node]
 5: 0xeb1685  [/home/user/.nvm/versions/node/v16.16.0/bin/node]
 6: 0xec134d v8::internal::Heap::CollectGarbage(v8::internal::AllocationSpace, v8::internal::GarbageCollectionReason, v8::GCCallbackFlags) [/home/user/.nvm/versions/node/v16.16.0/bin/node]
 7: 0xec404e v8::internal::Heap::AllocateRawWithRetryOrFailSlowPath(int, v8::internal::AllocationType, v8::internal::AllocationOrigin, v8::internal::AllocationAlignment) [/home/user/.nvm/versions/node/v16.16.0/bin/node]
 8: 0xe852c2 v8::internal::Factory::AllocateRaw(int, v8::internal::AllocationType, v8::internal::AllocationAlignment) [/home/user/.nvm/versions/node/v16.16.0/bin/node]
 9: 0xe7d8d4 v8::internal::FactoryBase<v8::internal::Factory>::AllocateRawWithImmortalMap(int, v8::internal::AllocationType, v8::internal::Map, v8::internal::AllocationAlignment) [/home/user/.nvm/versions/node/v16.16.0/bin/node]
10: 0xe7f5e0 v8::internal::FactoryBase<v8::internal::Factory>::NewRawOneByteString(int, v8::internal::AllocationType) [/home/user/.nvm/versions/node/v16.16.0/bin/node]
11: 0xe9239e v8::internal::Factory::NewStringFromOneByte(v8::base::Vector<unsigned char const> const&, v8::internal::AllocationType) [/home/user/.nvm/versions/node/v16.16.0/bin/node]
12: 0xd08745 v8::String::NewFromOneByte(v8::Isolate*, unsigned char const*, v8::NewStringType, int) [/home/user/.nvm/versions/node/v16.16.0/bin/node]
13: 0xbf607e  [/home/user/.nvm/versions/node/v16.16.0/bin/node]
14: 0xae6796  [/home/user/.nvm/versions/node/v16.16.0/bin/node]
15: 0x15878ec  [/home/user/.nvm/versions/node/v16.16.0/bin/node]
```

## BUG: ✘ [ERROR] fetch failed when publishing a workers-site #2245
Then run the following commands:
```
export NODE_OPTIONS=--max_old_space_size=40000
wrangler publish
```

You should get one of two errors.

Fetch failed:
```
Uploaded 0% (0 out of 15,317)
Uploaded 32% (5,000 out of 15,317)
Uploaded 65% (10,000 out of 15,317)
Uploaded 97% (15,000 out of 15,317)
Uploaded 100% (15,317 out of 15,317)

✘ [ERROR] fetch failed
```

Or too many bulk operations already in progress:
```
Uploaded 0% (0 out of 20,316)
Uploaded 24% (5,000 out of 20,316)
Uploaded 49% (10,000 out of 20,316)
Uploaded 73% (15,000 out of 20,316)
Uploaded 98% (20,000 out of 20,316)

✘ [ERROR] A request to the Cloudflare API (/accounts/***/storage/kv/namespaces/***/bulk) failed.

  Too many bulk operations already in progress. Please wait until one completes before retrying.
  [code: 10046]
  
  If you think this is a bug, please open an issue at:
  https://github.com/cloudflare/workers-sdk/issues/new/choose

```

Running wrangler publish again is often successful.

## BUG: Too many bulk operations already in progress
Uploading images (just to test things) seems to give the bulk operations in progress error.

In the Workers KV dashboard delete the KV namespace `__wrangler-kv-upload-failure-workers_sites_assets` so that there are no KV entries in the namespace.

Delete the current public folder with the website data.

Generate 100K images in the `public folder` with imagemagick:
```bash
mkdir public
cd public
for i in {1..100000}; do convert -size 128x128 xc: +noise Random jpg:random-$i.jpg; done
cd ..
```
It should take around 10 minutes to generate 100K files, which is about 2.6 GB.

Run the following commands:
```bash
export NODE_OPTIONS=--max_old_space_size=40000
wrangler publish
```

You should get this error on the first try:
```bash
✘ [ERROR] A request to the Cloudflare API (/accounts/***/storage/kv/namespaces/***/bulk) failed.

  Too many bulk operations already in progress. Please wait until one completes before retrying.
  [code: 10046]
  
  If you think this is a bug, please open an issue at:
  https://github.com/cloudflare/workers-sdk/issues/new/choose
```

Then on the second try it should work.
