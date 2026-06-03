# Gate 0: Data Feasibility — Findings & Decisions

## Phase 1: SoyAtlas Gene Annotation Version

### Gene ID Format
- **All 52,837 genes use Phytozome-style Glyma.XXGYYYYYY naming**
  - Example: `Glyma.01G000100`, `Glyma.13G257000`
- **Reference genome**: Gmax Wm82 (likely v2 or v4), same assembly as GCF_000004515.6
- 494 BioProjects, 19 tissue types
- Top tissues: root (157 BPs), leaf (137), seed (66), shoot (40)
- Rare tissues: epicotyl(1), petiole(1), radicle(1) — insufficient for grouped evaluation

### rnadata Gene ID System
- GFF has 54,262 `gene-LOC*` IDs + 707 `agat-gene-N` IDs
- matrix uses `agat-gene-N` format
- **Neither matches SoyAtlas Glyma.XXG naming**

### Gene ID Mapping Strategy
- Need Phytozome Wm82 annotation GFF with Glyma.XXG IDs → coordinate overlap with rnadata GFF
- Expected mapping rate: 70-85%
- Fallback: BLAST protein sequences

## Phase 2-5: (pending)

## Technical Decisions
| Decision | Rationale |
|----------|-----------|
| Use coordinate-based mapping via Phytozome annotation | Most reliable cross-reference for same-genome annotations |
| Exclude rare tissues from main evaluation | epicotyl/petiole/radicle have only 1 BP each |

## Resources
- SoyAtlas parquet_dir: 604 files, 52,837 genes, 5,481 samples
- rnadata: 260 samples, 58,427 genes, GFF_filtered.gff, transcripts.fa
- Both use GCF_000004515.6 (Gmax v4.0) genome assembly
