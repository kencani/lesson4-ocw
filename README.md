# 第四课作业 (源码在pallets/owc)

### 关键代码

#### 使用 unsigned tx with signed payload 的方式 ，保证安全而又无需手续费上链存储
```rust


		#[pallet::weight(10000)]
		pub fn submit_dot_price_unsigned_with_signed_payload(
			origin: OriginFor<T>,
			payload: PricePayload<T::Public>,
			_signature: T::Signature,
		) -> DispatchResult {
			let _who = ensure_none(origin)?;

			let PricePayload { price, public } = payload;

			log::info!("submit_dot_price_unsigned_with_signed_payload: ({:?}, {:?})", price, public);

			Prices::<T>::mutate(|prices| {
				if prices.len() == NUM_VEC_LEN {
					let _ = prices.pop_front();
				}
				prices.push_back(price);
			});

			Self::deposit_event(Event::NewPrice(price));
			Ok(())
		}


```

###  fetch_price_info 部分代码，通过对应api获取价格

```rust

fn fetch_price_info() -> Result<(), Error<T>> {

			let signer = Signer::<T, T::AuthorityId>::any_account();

			let mut lock = StorageLock::<BlockAndTime<Self>>::with_block_and_time_deadline(
				b"offchain-demo::dot-price::lock",
				LOCK_BLOCK_EXPIRATION,
				rt_offchain::Duration::from_millis(LOCK_TIMEOUT_EXPIRATION),
			);

			if lock.try_lock().is_err() {
				return Ok(());
			}

			let dot_price_info: DotPrice = Self::fetch_n_parse(DOT_PRICE_API)?;
			let mut price_iter = dot_price_info.price_to_usd.split(|char| *char == '.' as u8);

			let integer_part = Self::parse_float_part::<u64>(
				price_iter.next().ok_or(<Error<T>>::InvalidPriceIntegerValueError)?,
			)?;
			let fractional_part = Self::parse_float_part::<u32>(
				price_iter.next().ok_or(<Error<T>>::InvalidPriceValueFractionalError)?,
			)?;
			let fractional_part = Permill::from_parts(fractional_part);

			if let Some((_, res)) = signer.send_unsigned_transaction(
				|acct| PricePayload {
					price: (integer_part, fractional_part),
					public: acct.public.clone(),
				},
				Call::submit_dot_price_unsigned_with_signed_payload,
			) {
				return res.map_err(|_| {
					log::error!("Failed in submit_dot_price_unsigned_with_signed_payload");
					<Error<T>>::OffchainUnsignedTxSignedPayloadError
				});
			}

			Err(<Error<T>>::NoLocalAcctForSigning)
		}


```




###### 运行截图


![image](doc/1.png)


![image](doc/2.png)

