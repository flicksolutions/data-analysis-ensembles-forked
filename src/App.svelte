<script>
  import Chart from 'svelte-frappe-charts';
  import data from './assets/merged_argsEqPool5to10_1to2prem.csv'
  
  const divideByX = (data, x) => {
    let divided = [];
    const range = [...new Set(data.map(t => t[x]))]
    range.forEach( r => {
      divided = [...divided ,data.filter(p => {
          if (Array.isArray(p)){
            return p[0][x] === r
          } else {
            return p[x] === r
          }
        })
        ]
    })
    return divided
  }
  
  const divideByPool = x => {
    let nmInPoolSize = [];
    for(let c = 5; c<=10; c++){
      nmInPoolSize = [...nmInPoolSize ,x.filter(p => {
          if (Array.isArray(p)){
            return p[0].n_sentence_pool === `${c}`
          } else {
            return p.n_sentence_pool === `${c}`
          }
        })
        ]
    }
    return nmInPoolSize
  }

  const locals = data.filter(o => o.model_name === "LocalStandardReflectiveEquilibrium");
  const globals = data.filter(o => o.model_name === "GlobalNumpyReflectiveEquilibrium");
  const dataStructures = [...new Set(data.map(d => d.ds))];
  const pairs = globals.map(re => [re,locals.find(local => local.ds === re.ds && local.init_coms === re.init_coms)])
  const nm = pairs.filter(pair => (pair[0].fixed_point_coms !== pair[1].fixed_point_coms) || (pair[0].fixed_point_theory !== pair[1].fixed_point_theory))
  
  // convert strings to an array of integers
  const convert = (string, multiple) => {
    if(multiple) {
      return string.match(optRegEx).map(match => match.split(',').map(n => parseInt(n)))
    } else {
      return string.match(optRegEx)[0].split(',').map(n => parseInt(n))
    }
  }

  //create a object with Datastructeres as keys and their global optima (Pair of Theory, Commitment) as values
  const optima = {}
  const optRegEx = /(?!\{)[\d- \,]+(?=\})/g
  dataStructures.forEach(ds => {
    optima[ds] = globals.find(e => e.ds === ds)['global optima'].split('), ').map(ctPair => convert(ctPair, true))
  })

  //console.log(divideByPool(nm))
  const filteredData = divideByPool(nm);

  let achievement = filteredData.map((pool) => {
    const vs = {global: 0, local: 0, equal: 0}
    const regex = /((0|1)\.)\d+/g
    pool.forEach(pair => {
      const standardA = parseFloat(pair[0].achievements_evolution.match(regex).at(-1));
      const pieceMealA = parseFloat(pair[1].achievements_evolution.match(regex).at(-1));
      
      if (standardA > pieceMealA){
        if (pair[0].model_name === 'GlobalNumpyReflectiveEquilibrium') {
          vs.global++
        } else {
          vs.local++
        }
      } else if (standardA < pieceMealA) {
        if (pair[1].model_name === 'LocalStandardReflectiveEquilibrium') {
          vs.local++
        } else {
          vs.global++
        }
      } else if (standardA === pieceMealA) {
        vs.equal++
      }
    })
    return vs
  })

  /*
  // convert the absolute numbers into percentages of the rest of the dataset.
  */
  const getRelData = (domain, data) => {
    const sums = data.map(point => Object.values(point).reduce((a,b) => a + b))
    return data.map((collection,i) => collection[domain] / sums[i])
  }

  let fixPointChartData = {
    labels: [5,6,7,8,9,10],
    datasets: [
      {
        name: 'global',
        chartType: 'bar',
        values: getRelData('global', achievement)
      },
      {
        name: 'local',
        chartType: 'bar',
        values: getRelData('local', achievement)
      },
      {
        name: 'equal',
        chartType: 'bar',
        values: getRelData('equal', achievement)
      },
      /*show the sum as line in the same diagram... currently not possible beacause there is no option for multiple y axis. there is already an open PR... let's see if it will be approved soon.
      alternative: 100% auf 40. festsetzen!
      {
        name: 'sum',
        chartType: 'line',
        values: achievement.map(point => Object.values(point).reduce((a,b) => a + b))
      }*/
    ]
  }
  function compareArrays(arrayO,arrayR) {
    if(arrayO.length !== arrayR.length){
      return false
    }
    arrayO.forEach(e => {
      if (!arrayR.includes(e)){
        return false
      }
    })
    return true
  }

  //find out if a given re reached a global optimum
  const reachedOptimum = re => {
    const theoryToCheck = convert(re.fixed_point_theory,false);
    const comsToCheck = convert(re.fixed_point_coms,false)
    return optima[re.ds].some(optimum => {
      const optTheory = optimum[0]
      const optComs = optimum[1]
      if(compareArrays(optTheory,theoryToCheck)&&compareArrays(optComs,comsToCheck)) {
        return true
      }
    })
  }

  const reachedGlobal = [
      {
        name: 'global',
        values: divideByPool(globals).map(pool => {
          let counter = 0;
          pool.forEach(s => reachedOptimum(s) ? counter++:false)
          return counter
        })
      },
      {
        name: 'local',
        values: divideByPool(locals).map(pool => {
          let counter = 0;
          pool.forEach(s => reachedOptimum(s) ? counter++:false)
          return counter
        })
      }
]
const reachedGlobalChartData = {
  labels: [5,6,7,8,9,10],
  datasets: reachedGlobal
}
const getMean = pool => {
  return pool.map(pool => {
          const values = pool.map(r => parseInt(r.process_length));
          const sum = values.reduce((partial_sum, a) => partial_sum + a, 0);
          return sum/values.length
        })
}
const stepsChartData = {
  labels: [5,6,7,8,9,10],
  datasets: [
          {
        name: 'global',
        values: getMean(divideByPool(globals))
      },
      {
        name: 'local',
        values: getMean(divideByPool(locals))
      },
      {
        name: 'total',
        values: getMean(divideByPool(data))
      }
  ]
}

</script>

<main>
  <h1>Analyse Ensemblestudie</h1>

  <h2>Erkenntnisse:</h2>

  <ul>
    <li>RE-Prozesse: {data.length}</li>
    <li>Davon Local: {locals.length}</li>
    <li>Davon Global: {globals.length}</li>
    <li>Paare mit der selben Initials aber unterschiedlichen Fixpunkten: {nm.length}</li>
  </ul>
  <Chart title="höheres Achievement aufgeteilt nach Poolsize" data={fixPointChartData} type='axis-mixed' barOptions={{stacked: 1}} />
  <Chart title="erreichen eines Globalen Optimums aufgeteilt nach Poolsize" data={reachedGlobalChartData} type='line' />
  <Chart title="Durchschnitt an Schritten aufgeteilt nach Poolsize" data={stepsChartData} type='line' />
</main>

<style>
  :root {
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen,
      Ubuntu, Cantarell, 'Open Sans', 'Helvetica Neue', sans-serif;
  }

  main {
    padding: 1em;
    margin: 0 auto;
  }

  h1 {
    color: #ff3e00;
    text-transform: uppercase;
    font-size: 4rem;
    font-weight: 100;
    line-height: 1.1;
    margin: 2rem auto;
    max-width: 14rem;
  }

  @media (min-width: 480px) {
    h1 {
      max-width: none;
    }

    p {
      max-width: none;
    }
  }
</style>
