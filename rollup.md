# demo-rollup

代码分支 nightly

## transaction 

### sov-cli build sign send transaction

1. import a transaction from a JSON file at the provided path

``` rust
impl<Json, File> ImportTransaction<Json, File>
where
    Json: Subcommand,
    File: Subcommand,
{
    /// Parse from a file or a json string
    pub fn run<RT: CliWallet, C: sov_modules_api::Context, U, E1, E2, E3>(
        self,
        wallet_state: &mut WalletState<RT::Decodable, C>,
    ) -> Result<(), anyhow::Error>
    where
        Json: CliFrontEnd<RT> + CliTxImportArg,
        File: CliFrontEnd<RT> + CliTxImportArg,
        Json: TryInto<RT::CliStringRepr<U>, Error = E1>,
        File: TryInto<RT::CliStringRepr<U>, Error = E2>,
        RT::CliStringRepr<U>: TryInto<RT::Decodable, Error = E3>,
        RT::Decodable: BorshSerialize + BorshDeserialize + Serialize + DeserializeOwned,
        E1: Into<anyhow::Error> + Send + Sync,
        E2: Into<anyhow::Error> + Send + Sync,
        E3: Into<anyhow::Error> + Send + Sync,
    {
        let chain_id;
        let gas_tip;
        let gas_limit;

        let intermediate_repr: RT::CliStringRepr<U> = match self {
            ImportTransaction::FromFile(file) => {
                chain_id = file.chain_id();
                gas_tip = file.gas_tip();
                gas_limit = file.gas_limit();
                file.try_into().map_err(Into::<anyhow::Error>::into)?
            }
            ImportTransaction::FromString(json) => {
                chain_id = json.chain_id();
                gas_tip = json.gas_tip();
                gas_limit = json.gas_limit();
                json.try_into().map_err(Into::<anyhow::Error>::into)?
            }
        };

        let tx: RT::Decodable = intermediate_repr
            .try_into()
            .map_err(Into::<anyhow::Error>::into)?;

        let tx = UnsignedTransaction::new(tx, chain_id, gas_tip, gas_limit);

        println!("Adding the following transaction to batch:");
        println!("{}", serde_json::to_string_pretty(&tx)?);

        wallet_state.unsent_transactions.push(tx);

        Ok(())
    }
}
```

从文件或者字符串中解析出交易信息，构造未签名交易，放入 wallet_state 的 unsent_transactions 交易列表中。

2. Submit the Transaction

```rust
impl<C: sov_modules_api::Context + Serialize + DeserializeOwned + Send + Sync> RpcWorkflows<C> {
    /// Run the rpc workflow
    pub async fn run<Tx>(
        &self,
        wallet_state: &mut WalletState<Tx, C>,
        _app_dir: impl AsRef<Path>,
    ) -> Result<(), anyhow::Error>
    where
        Tx: Serialize + DeserializeOwned + BorshSerialize + BorshDeserialize,
    {
        // If the user is just setting the RPC url, we can skip the usual setup
        if let RpcWorkflows::SetUrl { rpc_url } = self {
            let _client = HttpClientBuilder::default()
                .build(rpc_url)
                .context("Invalid RPC url: ")?;
            wallet_state.rpc_url = Some(rpc_url.clone());
            println!("Set RPC url to {}", rpc_url);
            return Ok(());
        }

        // Otherwise, we need to initialize an  RPC and resolve the active account
        let rpc_url = wallet_state
            .rpc_url
            .as_ref()
            .ok_or(anyhow::format_err!(
                "No rpc url set. Use the `rpc set-url` subcommand to set one"
            ))?
            .clone();
        let client = HttpClientBuilder::default().build(rpc_url)?;
        let account = self.resolve_account(wallet_state)?;

        // Finally, run the workflow
        match self {
            RpcWorkflows::SetUrl { .. } => {
                unreachable!("This case was handled above")
            }
            RpcWorkflows::GetNonce { .. } => {
                let nonce = get_nonce_for_account(&client, account).await?;
                println!("Nonce for account {} is {}", account.address, nonce);
            }
            RpcWorkflows::GetBalance {
                account: _,
                token_address,
            } => {
                let BalanceResponse { amount } = BankRpcClient::<C>::balance_of(
                    &client,
                    None,
                    account.address.clone(),
                    token_address.clone(),
                )
                .await
                .context(BAD_RPC_URL)?;

                println!(
                    "Balance for account {} is {}",
                    account.address,
                    amount.unwrap_or_default()
                );
            }
            RpcWorkflows::SubmitBatch { nonce_override, .. } => {
                let private_key = load_key::<C>(&account.location)?;

                let nonce = match nonce_override {
                    Some(nonce) => *nonce,
                    None => get_nonce_for_account(&client, account).await?,
                };

                let txs = mem::take(&mut wallet_state.unsent_transactions)
                    .into_iter()
                    .enumerate()
                    .map(|(offset, tx)| {
                        Transaction::<C>::new_signed_tx(
                            &private_key,
                            tx.try_to_vec().unwrap(),
                            tx.chain_id,
                            tx.gas_tip,
                            tx.gas_limit,
                            nonce + offset as u64,
                        )
                        .try_to_vec()
                        .unwrap()
                    })
                    .collect::<Vec<_>>();

                let response: String = client
                    .request("sequencer_publishBatch", txs)
                    .await
                    .context("Unable to publish batch")?;

                // Print the result
                println!(
                    "Your batch was submitted to the sequencer for publication. Response: {:?}",
                    response
                );
            }
        }
        Ok(())
    }
}
```

从 wallet 中获取 private_key， 签名 wallet_state.unsent_transactions 中的未签名交易，生成签名交易。调用 json rpc 方法 sequencer_publishBatch 发送交易到节点。

### full node

1. verify transaction，send to da_service

``` rust
fn register_txs_rpc_methods<B, D>(
    rpc: &mut RpcModule<Sequencer<B, D>>,
) -> Result<(), jsonrpsee::core::Error>
where
    B: BatchBuilder + Send + Sync + 'static,
    D: DaService,
{
    rpc.register_async_method(
        "sequencer_publishBatch",
        |params, batch_builder| async move {
            let mut params_iter = params.sequence();
            while let Some(tx) = params_iter.optional_next::<Vec<u8>>()? {
                batch_builder
                    .accept_tx(tx)
                    .map_err(|e| to_jsonrpsee_error_object(e, SEQUENCER_RPC_ERROR))?;
            }
            let num_txs = batch_builder
                .submit_batch()
                .await
                .map_err(|e| to_jsonrpsee_error_object(e, SEQUENCER_RPC_ERROR))?;

            Ok::<String, ErrorObjectOwned>(format!("Submitted {} transactions", num_txs))
        },
    )?;
    rpc.register_method("sequencer_acceptTx", move |params, sequencer| {
        let tx: SubmitTransaction = params.one()?;
        let response = match sequencer.accept_tx(tx.body) {
            Ok(()) => SubmitTransactionResponse::Registered,
            Err(e) => SubmitTransactionResponse::Failed(e.to_string()),
        };
        Ok::<_, ErrorObjectOwned>(response)
    })?;

    Ok(())
}
```

sequencer_publishBatch 处理函数。

```rust
fn accept_tx(&mut self, raw: Vec<u8>) -> anyhow::Result<()> {
    if self.mempool.len() >= self.mempool_max_txs_count {
        bail!("Mempool is full")
    }

    if raw.len() > self.max_batch_size_bytes {
        bail!(
            "Transaction too big. Max allowed size: {}",
            self.max_batch_size_bytes
        )
    }

    // Deserialize
    let mut data = Cursor::new(&raw);
    let tx = Transaction::<C>::deserialize_reader(&mut data)
        .context("Failed to deserialize transaction")?;

    // Verify
    tx.verify().context("Failed to verify transaction")?;

    // Decode
    let msg = R::decode_call(tx.runtime_msg())
        .map_err(anyhow::Error::new)
        .context("Failed to decode message in transaction")?;

    self.mempool.push_back(PooledTransaction {
        raw,
        tx,
        msg: Some(msg),
    });
    Ok(())
}
```

校验交易，将交易放进交易池中。

``` rust
async fn submit_batch(&self) -> anyhow::Result<usize> {
    // Need to release lock before await, so the Future is `Send`.
    // But potentially it can create blobs that are sent out of order.
    // It can be improved with atomics,
    // so a new batch is only created after previous was submitted.
    tracing::info!("Submit batch request has been received!");
    let blob = {
        let mut batch_builder = self
            .batch_builder
            .lock()
            .map_err(|e| anyhow!("failed to lock mempool: {}", e.to_string()))?;
        batch_builder.get_next_blob()?
    };
    let num_txs = blob.len();
    let blob: Vec<u8> = borsh::to_vec(&blob)?;

    match self.da_service.send_transaction(&blob).await {
        Ok(_) => Ok(num_txs),
        Err(e) => Err(anyhow!("failed to submit batch: {:?}", e)),
    }
}
```
将交易发送给 DA 服务。

```rust
async fn send_transaction(&self, blob: &[u8]) -> Result<(), Self::Error> {
    debug!("Sending {} bytes of raw data to Celestia.", blob.len());

    let gas_limit = get_gas_limit_for_bytes(blob.len()) as u64;
    let fee = gas_limit * GAS_PRICE as u64;

    let blob = JsonBlob::new(self.rollup_batch_namespace, blob.to_vec())?;
    info!("Submitting: {:?}", blob.commitment);

    let height = self
        .client
        .blob_submit(
            &[blob],
            SubmitOptions {
                fee: Some(fee),
                gas_limit: Some(gas_limit),
            },
        )
        .await?;
    info!(
        "Blob has been submitted to Celestia. block-height={}",
        height,
    );
    Ok(())
}
```
