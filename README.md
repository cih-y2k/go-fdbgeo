# go-fdbgeo

[![GoDoc](https://godoc.org/github.com/umpc/go-fdbgeo?status.svg)](https://godoc.org/github.com/umpc/go-fdbgeo)
[![Go Report Card](https://goreportcard.com/badge/github.com/umpc/go-fdbgeo)](https://goreportcard.com/report/github.com/umpc/go-fdbgeo)

```sh
go get -u github.com/umpc/go-fdbgeo
```

This package contains tools for building geospatial layers using FoundationDB with Geohash-encoded keys.

This package uses its complimentary [`zrange`](https://github.com/umpc/go-zrange) package, for performing spatial range queries using FoundationDB with Geohash-encoded keys and a search radius.

The `RadialRange` method appears to be sufficient for range queries of around 5,000km or less. Changes that efficiently add support for larger query ranges are welcome [here](https://github.com/umpc/go-zrange).

**Note:** As of 5/26/2018, changes from [`apple/foundationdb` PR #408](https://github.com/apple/foundationdb/pull/408) must be applied to use this package, or a runtime panic will occur during subspace encoding.

## Example usage

```go
...

rangeParams := fdbgeo.RadialRangeParams{
  Subspace:  dir,
  Radius:    32.18688,
  Latitude:  37.334722,
  Longitude: -122.008889,
}

keyRanges := rangeParams.RadialRange()

fdbRangeOptions := fdb.RangeOptions{
  Mode: fdb.StreamingModeWantAll,
}

ret, err := db.ReadTransact(func(tr fdb.ReadTransaction) (ret interface{}, e error) {
  rangeResults := make([]fdb.RangeResult, len(keyRanges))

  for i, keyRange := range keyRanges {
    rangeResults[i] = tr.GetRange(keyRange, fdbRangeOptions)
  }

  var kvList []fdb.KeyValue
  for _, rangeResult := range rangeResults {
    ri := rangeResult.Iterator()

    for ri.Advance() {
      kv := ri.MustGet()

      // The following steps are only necessary when using WithinRadius before
      // value parsing.
      k, err := tuple.Unpack(kv.Key)
      if err != nil {
        panic(err)
      }

      var geohashID uint64
      switch hash := k[len(k)-2].(type) {
      case int64:
        geohashID = uint64(hash)
      case uint64:
        geohashID = hash
      }

      // Running this before value parsing may benefit performance, though it depends
      // on your data model.
      if !rangeParams.WithinRadius(geohashID) {
        continue
      }

      kvList = append(kvList, kv)
    }
  }

  return kvList, e
})

...
```

**Note:** Geohash range searches are imprecise. Results should be filtered using the input radius where precision is desired.

## References

* The `RadialRange` method was inspired by the algorithm in the "Search" section of [this page](https://web.archive.org/web/20180526044934/https://github.com/yinqiwen/ardb/wiki/Spatial-Index#search).
