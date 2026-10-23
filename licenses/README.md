# Extra license files

One crate that resvg and usvg are built from publishes only a file that names license texts the crate leaves out. The `crates/<crate>-<version>/` directory here holds those texts, and `scripts/cargo-licenses.py` adds them to the crate's own files in the image. The build stops when the binaries record a crate without any license file, and when they record another version of a crate that has a directory here, so that the text is checked again before the directory is renamed. A directory named without a version, `crates/<crate>/`, is kept across versions instead; it needs an `EXPRESSION` file with the crate's license expression, and the build stops when the crate's expression changes.

| Files | Source |
|---|---|
| `crates/siphasher-1.0.3/LICENSE-MIT`, `crates/siphasher-1.0.3/LICENSE-APACHE` | the texts that siphasher 1.0.3's `COPYING` names, from [jedisct1/rust-siphash](https://github.com/jedisct1/rust-siphash), where they were added in commit 07a910ae and first released in 1.0.4 |
