# Ingest SBoMs for releases

Part of the build process generates an SBoM for each release of a project. I want Imbi Gateway to ingest these and store the relationship between the release and associated software packages and versions. These will be surfaced as the Dependencies tab in the Imbi UI.

The translation of the SBoM into the Imbi database schema is handled by the Imbi API. The Imbi Gateway service will call the Imbi API to ingest the SBoM data. The Gateway is responsible for locating the Imbi Project associated with the incoming message and injecting the SBoM into the Imbi API as a `PUT` request with the project ID and release version in the HTTP resource URL.

The incoming SBoM data is expected to be in the format specified by version 1.5 of the [CycloneDX](https://cyclonedx.org/) standard.

I am currently generating SBoMs in this format using the following command for Javascript projects:

```bash
npm install --global '@cyclonedx/cyclonedx-npm@^1.19'
cyclonedx-npm --ignore-npm-errors --package-lock-only --flatten-components --short-PURLs \
              --omit dev --output-reproducible --spec-version 1.5 --mc-type application \
              --output-format JSON >"$RUNNER_TEMP/sbom.json"
```

For python packages, I am currently generating SBoMs in this format using the following command in a clean virtual environment with only the distributable package and its dependencies installed:

```bash
pip install cyclonedx-bom
python -m cyclonedx_bom environment \
	--pyproject "$working_dir/pyproject.toml" \
	--short-PURLs --spec-version 1.5 --output-format JSON \
	--no-validate --output-reproducible --output-file "$output" \
	$target_python
```

The process generates flattened SBoMs that contain only the components that are part of the distributable package. I did this to ensure consistency and reproducibility. I was also restricted to using PostgreSQL as the database backend without a graph extension so using a flattened representation was necessary. I would like to migrate to version 1.7 of the CycloneDX standard and use version 3.x of the CycloneDX Python library to parse SBoMs in the API.

## Modeling

I'm not completely sure how to best model the SBoM data in the database. My goal is to be able to answer the following types of questions by traversing the nodes and edges:

1. What components and versions are deployed in a given environment?
2. Which projects use a given component?
3. Which projects use a given version of a given component?

The word "use" in this context identifies the components and version that are associated with the specific versions of a project that is deployed in at least one environment.

I believe that we want to represent each component in an SBoM as a separate node in the graph. The component will have a "HAS_RELEASE" relationship to a version node. This is likely a new node class since it may have properties beyond the existing Release node. The component node will also have a "IDENTIFIED_BY" relationship to another node that represents the different unique identifier for the component such a package-urls or CPE IDs. I was previously using package URLs as the unique identifier for components. It is important that the link between a project release is to a specific version of a component, not just the component itself. This could be an edge to the component version node or an edge to the component node that is attributed by the specific version.

Graph traversal is important for answering the questions listed above. Specifically, a project release is associated with a specific environment. Introducing an edge between the project release node and the specific version of a component makes it possible to answer questions such as "What components and versions are deployed in a given environment?" and "Which projects use a given version of a given component?" without having to store any relationship between an environment and the component or version thereof.
