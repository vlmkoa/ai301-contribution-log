# Contribution [#1]: BWC tests failing for org.opensearch.knn.bwc.IndexingIT.testKNNDefaultIndexSettings

**Contribution Number:** 1
**Student:** Ryan Vo
**Issue:** [GitHub issue #1622](https://github.com/opensearch-project/k-NN/issues/1622)
**Status:** [Phase II] [Complete]

---

## Why I Chose This Issue

I chose this issue because this issue is at the intersection of distributed systems and low-level data handling, which is the area I want to grow in and have more experience. I understand that it is a backward compatibility failure in the k-NN plugin, where a search request serialized on a newer node cannot be correctly deserialized on an older node. I believe from this project, I can learn better how OpenSearch nodes communicate over the transport layer, how objects were versioned and serialized, and how rolling upgrades are tested.

It also seems interesting to me because it is labeled good first issue but is also non-trivial, as the maintainer @jmazanec15 investigated it and was "unable to reproduce so far." I hope to learn how to navigate a large, unfamiliar Java codebase, how OpenSearch handles serialization, and how to communicate findings clearly to maintainers.

---

## Understanding the Issue

### Problem Description

During a **rolling upgrade** — when an OpenSearch cluster is upgraded one node at a time, so old and new versions run side by side for a while — a k-NN search intermittently fails. The backward-compatibility (BWC) test `org.opensearch.knn.bwc.IndexingIT.testKNNDefaultIndexSettings` fails with an HTTP 500 and the error `unexpected byte [0x05]`. A search request that is serialized to bytes on one node is not read back correctly on another node that is running a different version.

### Expected Behavior

A `knn` query sent from one node should be deserialized correctly by every other node that receives it, regardless of which versions the two nodes are running during a rolling upgrade. The search should return results and the BWC test should pass consistently.

### Current Behavior

The search occasionally returns `500 Internal Server Error`. The server-side stack trace shows the failure happening while the receiving node reads the search request off the transport layer:

```
illegal_state_exception: unexpected byte [0x05]
  at StreamInput.readBoolean(StreamInput.java:593)
  at SearchSourceBuilder.<init>(SearchSourceBuilder.java:251)
  at ShardSearchRequest.<init>(ShardSearchRequest.java:244)
```

A `readBoolean` only accepts the bytes `0` or `1`; receiving `0x05` indicates the byte stream is being read at the wrong position. The issue is marked `flaky-test`, meaning it does not fail on every run — only intermittently.

### Affected Components

- **The transport-layer serialization of search requests** between mixed-version nodes (this is where the error surfaces, per the stack trace).
- **The k-NN query serialization path** — `KNNQueryBuilder` and its parser under `src/main/java/org/opensearch/knn/index/query/`, which encode/decode the `knn` query as part of the search request.
- **The rolling-upgrade BWC test suite** — `qa/rolling-upgrade/src/test/java/org/opensearch/knn/bwc/IndexingIT.java`.
- Note: the issue references PR #1620, but that PR was a C++/native-library packaging change, so it is not part of the Java serialization path under investigation.

---

## Reproduction Process

### Environment Setup

Setting up the build was the most challenging part, mostly because I'm on **Windows** and the project's developer guide targets Linux/macOS. I set up a Linux environment with **WSL2 (Ubuntu)** and worked entirely inside it. The errors I hit and how I solved them:

| Problem | Fix |
|--------|-----|
| SDKMAN install failed (`unzip` not found) | `sudo apt install -y unzip zip curl`, then installed JDK 21 via SDKMAN and set `JAVA21_HOME` (the test framework requires it) |
| `pip install cmake` blocked ("externally-managed-environment") | Installed CMake from apt instead (`sudo apt install -y cmake`); version satisfied the ≥3.24 requirement |
| CMake: "Could NOT find BLAS" | Installed Faiss native deps (`libopenblas-dev liblapack-dev libgflags-dev libomp-dev gfortran`). My bleeding-edge Ubuntu ships no *static* OpenBLAS, so I set `BLA_STATIC OFF` in `jni/cmake/init-faiss.cmake` to link the shared libraries instead |
| Build failed: "empty ident name" while applying native patches | Set `git config --global user.name/user.email` (the build commits the Faiss/nmslib patches) |
| Native library failed to load (`opensearchknn_faiss_avx2`) | WSL's CPU info is missing some AVX512 sub-flags, so the build produced a variant my runtime couldn't load. Fixed by building with `-Davx512.enabled=false -Davx512_spr.enabled=false`, which produces the AVX2 variant the runtime loads |
| Full `./gradlew build` hung at 94% | The full test suite stalls in my environment, so I used `./gradlew run` for the cluster and ran targeted tests instead of the whole suite |
| `gradle test --tests ...` failed with "No tests found" | Multi-project quirk: plain `test` runs in every subproject. Prefixing with the root project (`:test`) scopes it correctly |

End result: a working OpenSearch + k-NN cluster (`curl localhost:9200` returns the cluster info), built from my fork.

### Steps to Reproduce

1. Build and set up the plugin in WSL2 (see Environment Setup above).
2. Run the exact rolling-upgrade BWC test named in the issue, against the version pair the current code supports (2.20.0-SNAPSHOT → 3.7.0-SNAPSHOT):
   ```bash
   ./gradlew :qa:rolling-upgrade:testAgainstOneThirdUpgradedCluster \
     --tests "org.opensearch.knn.bwc.IndexingIT.testKNNDefaultIndexSettings" \
     -Dtests.bwc.version=2.20.0-SNAPSHOT \
     -Davx512.enabled=false -Davx512_spr.enabled=false \
     -Dnproc.count=2 --console=plain
   ```
   This builds the old (2.20) plugin, starts a 3-node old cluster, upgrades one node to 3.7 (a mixed cluster), and runs the search test.
3. **Observed result:** `BUILD SUCCESSFUL` — the test **passed**. The bug did **not** reproduce on the currently supported version pair.

### Reproduction Evidence

- **Branch:** https://github.com/vlmkoa/k-NN/tree/fix-issue-1622
- **My findings:**
  - The test **passes on current `main`** (2.20.0-SNAPSHOT → 3.7.0-SNAPSHOT), so the failure does not reproduce on the version pair the current code supports.
  - This is consistent with the issue being **flaky and version-specific**: it was originally reported between OpenSearch 3.0 (writing) and 2.14 (reading), versions that are no longer in the current BWC matrix, and the maintainer himself noted he was "unable to reproduce so far."
  - A single passing run does **not** prove the underlying problem is gone — it only shows it doesn't surface on this version pair. To witness the original failure I would need the historical version pair (which requires checking out the ~2024 state of the repo and JDK 17).
  - This "cannot reproduce on current versions" result is itself a useful, documentable finding for a stale flaky issue, and matches Step 3's note that an issue "may have been fixed upstream" or be environment/version-dependent.

---

## Solution Approach

> Reproduction is complete; the items below are my **plan for the next phase**. The detailed root-cause analysis and code change will be carried out and documented in Phase III.

### Analysis

The error is a **transport-layer serialization mismatch** between mixed-version nodes: bytes written by one version are read at the wrong offset by another version, which is what produces `unexpected byte [0x05]`. My next step is to trace the k-NN query's serialization path (`KNNQueryBuilder` and its parser) to pinpoint exactly where the encoding diverges across versions, and to confirm that hypothesis with a focused test before writing any fix.

### Proposed Solution

At a high level, I plan to make the `knn` query's serialization symmetric across versions so that a request written by one node is always read back identically by another, then add a regression test that simulates a mixed-version exchange so this can never silently regress again.

### Implementation Plan

Using the UMPIRE framework (adapted):

**Understand:** A k-NN search fails intermittently during rolling upgrades because a search request serialized on one node is misread by another node on a different version (`unexpected byte [0x05]`). It should round-trip correctly regardless of the version skew during an upgrade.

**Match:** Before coding, I will identify existing serialization round-trip tests (e.g. `KNNQueryBuilderTests`, `KNNQueryBuilderParserTests`) and the project's existing version-gating conventions in the query/parser code, and model my fix on whatever pattern the codebase already uses correctly elsewhere.

**Plan:**
1. Trace and confirm exactly where the `knn` query serialization diverges between versions.
2. Make the affected serialization handle version differences symmetrically (write and read must always agree).
3. Add a regression test that serializes a `knn` query under a simulated mixed-version scenario and asserts a clean round-trip.
4. Re-run the rolling-upgrade BWC test and the existing serialization tests to confirm nothing else breaks.

**Implement:** *(Phase III)* — work will happen on branch [`fix-issue-1622`](https://github.com/vlmkoa/k-NN/tree/fix-issue-1622); PR link to be added.

**Review:** Self-review against `CONTRIBUTING.md`:
- Sign every commit with DCO (`git commit -s`), using my **real name** and email (the guide forbids pseudonyms).
- Keep the Apache-2.0 license header on any new file.
- Add a `CHANGELOG.md` entry (CI enforces this for user-facing bug fixes).
- Run `./gradlew spotlessApply` / `spotlessJavaCheck` so formatting passes.
- Comment on the issue with my reproduction results and proposed approach before opening the PR, as the guide recommends.

**Evaluate:** See Testing Strategy below — a new regression test must pass only with the fix applied, the existing serialization tests must stay green, and the rolling-upgrade BWC test must remain green.

---

## Testing Strategy

### Unit Tests

- [ ] New regression test: serialize a `knn` query in a simulated mixed-version exchange and assert it deserializes to an identical object with no leftover bytes.
- [ ] Existing `KNNQueryBuilderTests` serialization round-trips continue to pass (no regression on same-version paths).
- [ ] Existing `KNNQueryBuilderParserTests` continue to pass.

### Integration Tests

- [ ] `qa:rolling-upgrade` — `IndexingIT.testKNNDefaultIndexSettings` stays green.
- [ ] Broader rolling-upgrade / restart-upgrade BWC suites are unaffected.

### Manual Testing

- Verified a local single-node cluster builds and runs (`./gradlew run`, then `curl localhost:9200` returns cluster info).
- Verified the BWC rolling-upgrade test executes end-to-end in my environment (currently passing on the supported version pair).

---

## Implementation Notes

### Week 1 Progress

- Set up the full build/dev environment in WSL2 and documented every error + fix (see Environment Setup).
- Created and pushed the working branch `fix-issue-1622` to my fork.
- Reproduced (attempted) the failing BWC test on the current supported version pair; documented that it currently passes and why that's consistent with a flaky, version-specific issue.
- Drafted the solution plan and testing strategy for Phase III.

### Code Changes

- **Files modified:** _(none yet — Phase III)_
- **Key commits:** _(to be added)_
- **Approach decisions:** _(to be added)_

---

## Pull Request

**PR Link:** _(to be added in Phase III)_

**PR Description:** _(draft to be added)_

**Maintainer Feedback:**
- _(to be added)_

**Status:** Not yet submitted (planning complete)

---

## Learnings & Reflections

### Technical Skills Gained

- Building a mixed Java + C++/JNI project from source, including native dependency management (BLAS/LAPACK/OpenMP) and CPU-instruction-set (AVX) build flags.
- How OpenSearch runs rolling-upgrade BWC tests across multiple node versions.

### Challenges Overcome

- A long chain of environment/setup failures on Windows/WSL — solved one error at a time and documented each fix.

### What I'd Do Differently Next Time

- _(to be added at the end of the contribution)_
