# htsjdk CRAM 3.1 support

## Read support

Read support for CRAM 3.1 is available in the 3.4.0 release. This will hopefully be minted as a GitHub release in the coming few days. See samtools/htsjdk#1755.

## Write support

The following summaries has been generated using Claude Opus over [htsjdk](https://github.com/broadinstitute/htsjdk), [htslib](https://github.com/samtools/htslib), [noodles](https://github.com/noodles-io/noodles) and [samtools](https://github.com/samtools/samtools) source code. The content here has not been fully verified and should be reviewed before using.


### Phase 1: Basic CRAM 3.1 Writing (Normal Profile)

See [phase1-cram31-write-plan.md](phase1-cram31-write-plan.md) for more information.

The vast majority of the work for basic CRAM 3.1 writing already exists on the `cn_cram_3_1_write` branch. Merging this branch removes the hard block in `CRAMEncoderV3_1`, changes the default output version to 3.1, and switches the encoding map from rANS 4x8 to rANS Nx16 with Name Tokeniser for read names. This is equivalent to htslib's `normal` profile and requires no new codec implementations — it is purely wiring up encoders that already work. Quality scores remain compressed with rANS Nx16 order-1 rather than FQZComp.

### Phase 2: FQZComp Encoding

See [phase2-fqzcomp-encode-plan.md](phase2-fqzcomp-encode-plan.md) for more information.

FQZComp is a context-adaptive quality score compressor and is the single biggest piece of engineering remaining. All the low-level encoding primitives (`ByteModel.modelEncode()`, `RangeCoder`) already exist in htsjdk from the decoder side, so the work is in orchestrating these into a working encoder and choosing sensible default parameters. The encoder needs to serialise its parameter header (quality maps, context tables, global flags), then encode each quality byte through the adaptive model array indexed by a 16-bit context. `CRAMCodecModelContext`, which is already threaded through the write path but currently empty, needs populating with per-read metadata (at minimum read lengths). A single hardcoded preset matching htslib's default level is a reasonable starting point; additional presets for the `small` and `archive` profiles can follow.

### Phase 3: Stripe Encoding and Encoding Profiles

See [phase3-stripe-and-profiles-plan.md](phase3-stripe-and-profiles-plan.md) for more information.

Stripe is the last unimplemented transform flag for both rANS Nx16 and Range coding. The algorithm is straightforward — transpose input into N byte-columns, compress each independently, concatenate — and both decoders already exist as reference implementations. With stripe and FQZComp in place, htsjdk has the full CRAM 3.1 codec feature set, and we can introduce encoding profiles (`fast`, `normal`, `small`, `archive`) matching htslib/samtools. This means adding an `EncodingProfile` enum to `CRAMEncodingStrategy`, making `CompressionHeaderEncodingMap` profile-aware, and surfacing the choice through `CRAMEncoderOptions` and `SAMFileWriterFactory`. All the underlying compressors (BZIP2, LZMA, Range) are already implemented — the work is selection logic and wiring. A useful detail: the `fast` profile should produce CRAM 3.0 files since it only uses GZIP, matching the spec requirement that files using only codecs 0-4 be written as 3.0 for backward compatibility.