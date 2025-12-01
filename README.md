<img style="margin: 0px" alt="Repository Header Image" src="./resources/repo-header.png" />

# Requests For Comments for the St. Jude Rust Labs project

This repo structure and contents are ~~stolen~~ borrowed from the [St. Jude Cloud Team's `rfcs` repo](https://github.com/stjudecloud/rfcs).

These RFCs are meant to act as a public archive of our design intentions and to facilitate conversation both amongst the internal team and any external parties who may wish to help shape the future of the Sprocket project. Notably, these documents are not authoritative in any fashion. They are a snapshot of design goals; some details are important to hash out ahead of time, but practical experience with an implementation or changing understanding of a problem space as learned through use of a feature will likely cause deviations from the initial plans as laid out in these RFCs.

We will not make an effort to backport changes to these documents if we feel the conversations have run their course.

## Install

```sh
cargo install mdbook
```

## Usage

```sh
mdbook build
python3 -m http.server -d book
# visit the rendered version in your browser at http://localhost:8000.
```

## 📝 License and Legal

This project is licensed as either [Apache 2.0][license-apache] or
[MIT][license-mit] at your discretion. Additionally, please see [the
disclaimer](https://github.com/stjude-rust-labs#disclaimer) that applies to all
crates and command line tools made available by St. Jude Rust Labs.

Copyright © 2023-Present [St. Jude Children's Research Hospital](https://github.com/stjude).

[license-apache]: https://github.com/stjude-rust-labs/rfcs/blob/main/LICENSE-APACHE
[license-mit]: https://github.com/stjude-rust-labs/rfcs/blob/main/LICENSE-MIT
