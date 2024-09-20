## Running dify at apolo MLOps platform

###  tl; dr:

After clonning this repository, you will need to `apolo-flow run` jobs in a following order:
1. `pgvector` (RDBMS and vector store components are used).
2. `redis`
3. `api` (update `jobs.api.env.INIT_PASSWORD` to your `secret:` value. this pass allows creating admin account)
4. `worker`
5. `web` 
6. `nginx` (this one will automatically open web interface in your default browser)

Afterwards, the application is ready to be used.

In order to create first admin account, click on "Setting up an admin account". Use `jobs.api.env.INIT_PASSWORD` value to authenticate yourself. 

NOTE: default lifespan is configured to be 10 days. Afterwards jobs will terminate. You could `apolo job bump-life-span <job-name-or-id> Nd` to bump specific job's lifespan for extra N days. Alternatively, you could also change `defaults.life_span` value in [life.yaml](.neuro/live.yaml) file to needed value.

#### Integrating chat LLM
Spinup [vLLM inference server](https://docs.vllm.ai/en/latest/).

It could be either Apolo app, or this flow's `vllm` job.

For `vllm` job, adjust model name (if needed) and `apolo secret add HF_TOKEN` -- your [huggingface access token](https://huggingface.co/docs/hub/en/security-tokens).

Afterwards, within Dify, integrate it. In order to do that, click on your username in top right corner -> settings -> model provider. Select "OpenAI-API-compatible" and click "add". Select "LLM" radiobutton, model name equals to the one you specified in vLLM. API key is empty (or one that you specified for vLLM inference server). API endpoint URL is vLLM job's **internal hostname named** (see in output of `apolo-flow status vllm`). Example: `http://vllm--0e1436ea6a.platform-jobs:8000/v1`. Context size and number of depends on particular LLM. Llama 3.1 has 128k context size. Max tokens depends on amount of memory the underlying GPU have.  

#### Integrating embedding inference server
Spinup [text-embeddings-inference](https://github.com/huggingface/text-embeddings-inference).

The same here, it could be either Apolo app, or this flow's `tei` job.

Navigate to the same page where you added OpenAI-API-compatible LLM inference, but this time pick "text-embeddings-inference". Enter model name and internal hostname of the server with port. Example: `http://text-embeddings-inference--0e1436ea6a.platform-jobs:3000`.

Congratulations! Now you are ready to use Dify.


<details> 

<summary>Diffy components FAQ</summary>

1. `apolo-flow run pgvector` creates instance of PostgreSQL server with PGVector extension installed. The database is used to store metadata for your RAGs, datasets and configurations of apps within Dify.

It might happen, that PGVector crashes on startup due to inability to obtain file locks. This is caused by implementation of underlying storage provider for apolo files app (used in volumes). To overcome this, create apolo disk of needed size (`apolo disk create --help`) and replace `volumes.pgdata.remote` with reference to the disk ID. The processed datasets' vectors are stored here too, therefore pick disk size carefully.

The updated config should look like:

```yaml
volumes:
    pgvector:
        remote: disk:disk-8f7e821d-94a7-4ae2-aece-3d115c698268 # you could refer disk by name too
```

2. `apolo-flow run redis` creates instance of Redis standalone server. It is used by Dify API to schedule data processing tasks onto dify workers via Celery.

The same issue might happen here as with PostgreSQL. In this case, create another dedicated apolo disk and update `volumes.redis.remote` in live.yaml config file.

3. `apolo-flow run api` starts Dify API job. This is Dify's main controller.
4. `apolo-flow run worker` starts Dify worker that processes datasets and other tasks that you might execute within the app. You could start several of them.
5. `apolo-flow run web` starts web server job that hosts web application with those fancy forms and controllers. It communicates with the API.
6. `apolo-flow run nginx` starts NGINX reverse proxy job that acts as a facade for Dify's API and web server components. You will open web interface of Dify via nginx. After starting this job, a corresponding tab within your web browser should open.

Do not hesitate to contact Apolo support if something goes wrong. We are eager to help you build awesome RAGs!
</details>