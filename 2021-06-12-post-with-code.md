# Code Example

This post contains code.

```rust
#[post("/", data = "<paste>")]
fn upload(paste: Data, config: State<Config>) -> Result<String, Status> {
    let ds = paste.open();
    let mut buffer = Vec::new();
    let limit = (config.maxsize + 1) as u64;

    // Read up to maxsize + 1 bytes
    let mut handle = ds.take(limit);
    let n = handle
        .read_to_end(&mut buffer)
        .map_err(|_| Status::InternalServerError)?;
    if n > config.maxsize {
        return Err(Status::PayloadTooLarge);
    }

    let paste_id = PasteId::new(5);
    let filename = format!("{}/{}", config.data_dir, paste_id);

    let mut file = File::create(&filename).map_err(|_| Status::InternalServerError)?;
    file.write_all(&buffer)
        .map_err(|_| Status::InternalServerError)?;

    let url = format!("{}/{}\n", config.host, paste_id);
    Ok(url)
}
```
