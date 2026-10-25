# iLabs TM-T20II driver

CUPS driver for the **Epson TM-T20II** thermal receipt printer on Linux, packaged with a
known-good configuration.

This is a derivative of [klirichek/zj-58](https://github.com/klirichek/zj-58) — an ESC/POS
`rastertozj` CUPS filter for cheap thermal receipt printers. Upstream supports the TM-T20
already; this repo pins down the parts that are easy to get wrong and documents the working
setup end to end. Original upstream README is preserved as [UPSTREAM-README.md](UPSTREAM-README.md).

## Why this repo exists

Two things bite people setting up a TM-T20II, and neither produces an obvious error message:

**1. The PPD alone does nothing.** A PPD is just a description file. It declares

```
*cupsFilter: "application/vnd.cups-raster 100 rastertozj"
```

which means CUPS hands every job to a binary named `rastertozj` that has to be compiled and
installed separately. It is not packaged in Debian/Ubuntu. Without it, CUPS accepts jobs and
kills them instantly:

```
File "/usr/lib/cups/filter/rastertozj" not available: No such file or directory
Alerts: cups-missing-filter-warning
```

Note that `printer-driver-foo2zjs` is **not** the missing piece — that is an unrelated ZjStream
driver for Zenographics laser printers and provides no `rastertozj`.

**2. `*cupsModelNumber` is the print head width in pixels, not a model ID.**

```c
int iMaxWidth = settings.modelNum;   // rastertozj.c
```

| Paper | Pixels |
|-------|--------|
| 58 mm | 384    |
| 80 mm | 560    |

The TM-T20II is an 80 mm printer, so it must be **560**. Old forks circulate a `tm20.ppd`
containing `*cupsModelNumber: 20`, which the current filter reads as a 20-pixel-wide head. The
`tm20.ppd` in this repo is generated from `zjdrv.drv` by `ppdc` and has the correct value.

## Build

Upstream uses CMake, which works fine. But it is a single C file, so plain gcc is enough:

```bash
gcc -O2 -Wall -fPIC -o rastertozj rastertozj.c -lcups
```

Requires the CUPS development headers: `libcups2-dev` on Debian/Ubuntu, `libcups` on Arch.

On CUPS 2.4 the raster API — `cupsRasterOpen`, `cupsRasterReadPixels` and friends — lives in
`libcups.so.2`, so `-lcups` is all it needs. On older CUPS add `-lcupsimage`. Verify with
`nm -D --undefined-only rastertozj | grep -i raster` and a `LD_BIND_NOW=1` run.

The CMake route, if you prefer it:

```bash
mkdir build && cd build && cmake .. && cmake --build .
```

## Install

```bash
sudo install -o root -g root -m 755 rastertozj /usr/lib/cups/filter/rastertozj
sudo install -o root -g root -m 644 -D tm20.ppd /usr/share/cups/model/zj-58/tm20.ppd
sudo systemctl restart cups
```

Then point the queue at the PPD:

```bash
lpadmin -p TM-T20II -P /usr/share/cups/model/zj-58/tm20.ppd
```

CUPS admin commands (`lpadmin`, `cupsaccept`, `cancel`) do not need `sudo` if your user is in
the `lpadmin` group — cupsd accepts peer credentials over the local socket.

If you are upgrading from an older zj-58, you **must** explicitly re-select the PPD as above.
CUPS keeps using its cached copy in `/etc/cups/ppd/` otherwise, and an old-format PPD will fail
against the current filter.

## Cutter and feed configuration

```bash
lpadmin -p TM-T20II \
  -o OptionCutter=True \
  -o CutMedia=EndOfJob \
  -o FeedWhere=AfterJob \
  -o FeedDist=6feed21mm
```

The feed distance matters. The autocutter blade sits roughly **16 mm downstream of the print
head**, so cutting immediately after the last raster line slices through the bottom of the
receipt. The driver's default is only 9 mm, which still clips. Feed distance should be

> blade offset (~16 mm) + desired bottom margin

`6feed21mm` gives about 5 mm of clear paper below the text. Options run from 3 mm to 45 mm in
3 mm steps. The filter already feeds before it cuts, so this is purely a distance question:

```c
if (settings.AdvanceMedia == CUPS_ADVANCE_JOB)
  flushManyLines(settings.AdvanceDistance);

if (settings.CutMedia == CUPS_CUT_JOB)
  cutMedia();
```

`CutMedia=EndOfJob` cuts once per job; `EndOfPage` cuts between every page of a multi-page job.

## Verifying

```bash
lpstat -l -p TM-T20II          # want: Alerts: none
lpoptions -p TM-T20II -l       # '*' marks the active default
printf 'test\n\n' | lp -d TM-T20II
```

A healthy run leaves no `E` lines in `/var/log/cups/error_log`.

## Tested on

Ubuntu 24.04, CUPS 2.4.7, Epson TM-T20II over LPD
(`dnssd://EPSON%20TM-T20II%20(628DEA)._printer._tcp.local/`), 80 mm roll, 203 dpi.

## License

BSD 3-Clause, unchanged from upstream — Copyright (c) 2014, Aleksey N. Vinogradov.
See [LICENSE](LICENSE). `rastertozj.c`, `zjdrv.drv` and `CMakeLists.txt` are upstream work.
