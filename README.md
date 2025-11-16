# gh-star

<div markdown="1" style="text-align: center; font-size: 1.2em;">

<b>🚧 Fork in progress, expect some dust 🚧</b>

[![github-tag](https://img.shields.io/github/v/tag/serpro69/gh-star?style=for-the-badge&logo=semver&logoColor=white)](https://github.com/serpro69/gh-star/tags)
[![github-license](https://img.shields.io/github/license/serpro69/gh-star?style=for-the-badge&logo=unlicense&logoColor=white)](https://opensource.org/license/mit)
[![github-stars](https://img.shields.io/github/stars/serpro69/gh-star?logo=github&logoColor=white&color=gold&style=for-the-badge)](https://github.com/serpro69/gh-star)

</div>

`gh-star` is a [github cli extension](https://docs.github.com/en/github-cli/github-cli/creating-github-cli-extensions#about-github-cli-extensions) for working with your personal github Stars and Star Lists from your terminal.

## Installation

Install the latest release using the GitHub CLI:

```bash
gh extension install serpro69/gh-star
```

### Installation from Source

```bash
git clone https://github.com/serpro69/gh-star.git
cd gh-star && go build . && gh extension install .
```

> [!TIP]
> Clone the repo to a permanent location, gh cli will alias the binary when you run `gh extension install .` command.
