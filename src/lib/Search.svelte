<script>
  import AutoComplete from './Autocomplete.svelte'
  import { location } from '../store.js'

  function jsonToURL(obj) {
    return Object.keys(obj)
      .map(i => i + '=' + encodeURIComponent(obj[i]))
      .join('&')
  }

  let items = []
  let nomatch = false
  let spinner = false

  // set store to selected value
  function handleHit(event) {
    location.set({
      label: event.detail.value,
      address: event.detail.address,
      latlng: [Number(event.detail.lat), Number(event.detail.lng)],
      pid: event.detail.pid,
      groundpid: event.detail.groundpid
    })
  }

  // fetch suggestions
  async function handleQuery(event) {
    const queryString = event.detail.trim()
    const urls = []

    // address
    const addressArg = {
      columns: "full_address as value, 'ADDRESS' as type, groundpid, round(ST_X(ST_Transform(the_geom, 4326))::NUMERIC,4) as lng, round(ST_Y(ST_Transform(the_geom, 4326))::NUMERIC,4) as lat, num_parent_parcel as pid, full_address as address",
      limit: 8,
      filter: `ts @@ to_tsquery('addressing_en', '${queryString.toUpperCase().replace(/ /g, '&') + ':*'}') and cde_status='A' and the_geom is not null`
    }
    urls.push(`https://maps.mecklenburgcountync.gov/n2/api/v1/query/master_address_table?${jsonToURL(addressArg)}`)

    // parks
    const parkArg = {
      columns: `prkname as value, 'PARK' as type, round(ST_X(ST_Transform(p.the_geom, 4326))::NUMERIC,4) as lng, round(ST_Y(ST_Transform(p.the_geom, 4326))::NUMERIC,4) as lat, t.pid as pid, prkaddr as address`,
      limit: 5,
      filter: `prkname ilike '%${queryString}%' and p.the_geom && t.the_geom`
    }
    urls.push(`https://maps.mecklenburgcountync.gov/n2/api/v1/query/parks p, tax_parcels t?${jsonToURL(parkArg)}`)

    // libraries
    const libraryArg = {
      columns: `name as value, 'LIBRARY' as type, round(ST_X(ST_Transform(l.the_geom, 4326))::NUMERIC,4) as lng, round(ST_Y(ST_Transform(l.the_geom, 4326))::NUMERIC,4) as lat, p.pid as pid, address`,
      limit: 5,
      filter: `name ilike '%${queryString}%' and l.the_geom && p.the_geom`
    }
    urls.push(`https://maps.mecklenburgcountync.gov/n2/api/v1/query/libraries l, tax_parcels p?${jsonToURL(libraryArg)}`)

    // pid
    if (!isNaN(queryString) && queryString.length >= 7) {
      const pidArg = {
        columns: `num_parent_parcel as value, 'PARCEL' as type, groundpid, round(ST_X(ST_Transform(the_geom, 4326))::NUMERIC,4) as lng, round(ST_Y(ST_Transform(the_geom, 4326))::NUMERIC,4) as lat, num_parent_parcel as pid, full_address as address`,
        limit: 5,
        filter: `num_parent_parcel like '${queryString}%' and the_geom is not null and cde_status='A'`
      }
      urls.push(`https://maps.mecklenburgcountync.gov/n2/api/v1/query/master_address_table?${jsonToURL(pidArg)}`)
    }

    // Fetch all the things
    spinner = true
    Promise.all(urls.map(url =>
      fetch(url)
        .then(resp => resp.json())
    )).then(jsons => {
      spinner = false
      nomatch = false
      items = [].concat(...jsons).map(elem => {
        elem.groundpid = elem.groundpid || elem.pid
        return elem
      })
      if (items.length === 0) nomatch = true
    })

  }
  </script>


  <AutoComplete placeholder="Try '2145 Suttle' or 'Jetton'" minChar="4" nomatch={nomatch} {items} on:hit={handleHit} on:query={handleQuery} value={$location.address} spinner={spinner} />
