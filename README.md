# Shopware Developer Docs & LangChain

This implementation uses the [LangChain](https://langchain.readthedocs.io/en/latest/index.html) library.

It is based on the [LangChain Chat](https://blog.langchain.dev/langchain-chat/) implementation.

## Setup

In order to use it, clone the repo and [install Jupyter Notebook](https://jupyter.org/install#jupyter-notebook) (Python required) on your machine.

Then run

```sh
jupyter notebook
```

in the repo root directory.

## Usage

Open the notebook file `docs-ingest-evaluation.ipynb` and execute the steps one by one from top to bottom. The step "create vector store" takes about 10 minutes, but it only has to be performed once (unless you make changes to the preceding steps of course).