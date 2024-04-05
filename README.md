# AlertFlow

This project contains the Airflow configuration for [AlertaDengue](https://info.dengue.mat.br). To run it, you will
need:

- [make](https://www.gnu.org/software/make/)
- [Docker compose](https://docs.docker.com/compose/)

## Local Setup

To run locally, you will need to:
- Generate the `.env` file (use `env.tpl` as a template)
- Create an account on [Copernicus Climate](https://cds.climate.copernicus.eu/). After creating the account, copy the
  user ID and the API key into the variable `CDSAPI_KEY=userid:apikey` in your `.env`.
- Execute `make containers-build`
- Execute `make containers-start`
- Access [localhost:8081](http://localhost:8081/)
- The DAGs will be in `alertflow/dags`

## Deployment

We use [Dokku](https://dokku.com/) for deployment. Check out [docs/deploy-dokku.md](docs/deploy-dokku.md) for more
info.
