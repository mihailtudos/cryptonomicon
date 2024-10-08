<script>
export default {
  name: 'App',
  data() {
    return {
      ticker: null,
      tickers: [],
      sel: null,
      graph: [],
      allTickers: [],
      errorMessage: null,
      page: 1,
      filter: '',
      hasNextPage: true
    }
  },
  created: async function () {
    const windoData = Object.fromEntries(new URL(window.location).searchParams.entries())

    if (windoData.filter) {
      this.filter = windoData.filter
    }
    if (windoData.page) {
      this.page = windoData.page
    }

    const tickersData = localStorage.getItem('cryptonomicon-list')
    if (tickersData) {
      this.tickers = JSON.parse(tickersData)
      this.tickers.forEach((ticker) => this.subscribeToUpdates(ticker.name))
    }
    try {
      const response = await fetch(
        'https://min-api.cryptocompare.com/data/all/coinlist?summary=true'
      )
      const { Data } = await response.json()
      if (response.status !== 200) throw new Error('Something went wrong')
      Object.keys(Data).forEach((k) => {
        this.allTickers.push({
          symbol: k,
          fullName: Data[k].FullName
        })
      })
    } catch (error) {
      console.log(error.message)
    }
  },
  methods: {
    subscribeToUpdates(tickerName) {
      setInterval(async () => {
        const f = await fetch(
          `https://min-api.cryptocompare.com/data/price?fsym=${tickerName}&tsyms=USD&api_key=45213c0dff5c7b67a8f6f76ba0e18bf4c188727f78c3aae9a9b60d1b3c59a03f`
        )
        const data = await f.json()
        this.tickers.find((t) => t.name === tickerName).price =
          data.USD > 1 ? data.USD.toFixed(2) : data.USD?.toPrecision(2)

        if (this.sel?.name === tickerName) {
          this.graph.push(data.USD)
        }
      }, 3000)
    },
    add(option) {
      if (option) {
        this.ticker = option
      }

      const newTicker = {
        name: this.ticker.toUpperCase(),
        price: '-'
      }

      const idx = this.tickers.findIndex((t) => t.name == this.ticker)
      if (idx != -1) {
        this.errorMessage = 'Ticker already added'
        this.ticker = null
        return
      }
      this.subscribeToUpdates(newTicker.name)
      this.tickers.push(newTicker)
      this.filter = ''
      localStorage.setItem('cryptonomicon-list', JSON.stringify(this.tickers))
      this.ticker = null
    },
    filteredTickers() {
      const start = (this.page - 1) * 6
      const end = this.page * 6
      const filteredTickers = this.tickers.filter((ticker) =>
        ticker.name.includes(this.filter.toUpperCase())
      )

      this.hasNextPage = filteredTickers.length > end

      return filteredTickers.slice(start, end)
    },
    normalizeGraph() {
      const maxValue = Math.max(...this.graph)
      const minValue = Math.min(...this.graph)
      console.log(maxValue, minValue)

      return this.graph.map((price) => 5 + ((price - minValue) * 95) / (maxValue - minValue))
    },
    select(ticker) {
      this.sel = ticker
      this.graph = []
    },
    handleDelete(tickerToDelete) {
      this.tickers = this.tickers.filter((t) => t !== tickerToDelete)
    }
  },
  computed: {
    suggestedTickers() {
      const options = []
      if (this.ticker && this.allTickers.length) {
        this.allTickers.forEach((t) => {
          if (
            (t.symbol.includes(this.ticker.toUpperCase()) ||
              t.fullName.includes(this.ticker.toUpperCase())) &&
            options.length < 5
          ) {
            options.push(t.symbol)
          }
        })
      }

      return options
    }
  },
  watch: {
    filter() {
      console.log('called')
      this.page = 1
      const { pathname } = window.location

      window.history.pushState(
        null,
        document.title,
        `${pathname}?filter=${this.filter}&page=${this.page}`
      )
    },
    page() {
      const { pathname } = window.location

      window.history.pushState(
        null,
        document.title,
        `${pathname}?filter=${this.filter}&page=${this.page}`
      )
    }
  }
}
</script>

<template>
  <div
    class="mx-auto flex flex-col items-center justify-between h-full min-h-screen bg-gray-100 p-4"
  >
    <!-- <div
      class="fixed w-100 h-100 opacity-80 bg-purple-500 inset-0 z-50 flex items-center justify-center"
    > -->
    <!-- <div
      class="fixed w-100 h-100 opacity-80 flex items-center justify-center"
    >
      <svg
        class="animate-spin -ml-1 mr-3 h-12 w-12 text-white"
        xmlns="http://www.w3.org/2000/svg"
        fill="none"
        viewBox="0 0 24 24"
      >
        <circle
          class="opacity-25"
          cx="12"
          cy="12"
          r="10"
          stroke="currentColor"
          stroke-width="4"
        ></circle>
        <path
          class="opacity-75"
          fill="currentColor"
          d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"
        ></path>
      </svg>
    </div> -->
    <div class="w-full flex flex-col justify-between items-start">
      <section>
        <div class="flex">
          <div class="max-w-xs">
            <label for="wallet" class="block text-sm font-medium text-gray-700">Ticker</label>
            <div class="mt-1 relative rounded-md shadow-md">
              <input
                @keydown.enter="add"
                v-model="ticker"
                type="text"
                name="wallet"
                id="wallet"
                class="block w-full px-4 border-gray-300 text-gray-900 focus:outline-none focus:ring-gray-500 focus:border-gray-500 sm:text-sm rounded-md"
                placeholder="e.g. DOGE"
              />
            </div>
            <div
              v-if="suggestedTickers.length"
              class="flex bg-white shadow-md p-1 rounded-md shadow-md flex-wrap"
            >
              <span
                v-for="option in suggestedTickers"
                :key="option"
                @click="add(option)"
                class="inline-flex items-center px-2 m-1 rounded-md text-xs font-medium bg-gray-300 text-gray-800 cursor-pointer"
              >
                {{ option }}
              </span>
            </div>
            <div v-if="errorMessage" class="text-sm text-red-600">{{ errorMessage }}</div>
          </div>
        </div>
        <button
          @click="add"
          type="button"
          class="my-4 inline-flex items-center py-2 px-4 border border-transparent shadow-sm text-sm leading-4 font-medium rounded-full text-white bg-gray-600 hover:bg-gray-700 transition-colors duration-300 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-gray-500"
        >
          <!-- Heroicon name: solid/mail -->
          <svg
            class="-ml-0.5 mr-2 h-6 w-6"
            xmlns="http://www.w3.org/2000/svg"
            width="30"
            height="30"
            viewBox="0 0 24 24"
            fill="#ffffff"
          >
            <path
              d="M13 7h-2v4H7v2h4v4h2v-4h4v-2h-4V7zm-1-5C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm0 18c-4.41 0-8-3.59-8-8s3.59-8 8-8 8 3.59 8 8-3.59 8-8 8z"
            ></path>
          </svg>
          Add crypto
        </button>
      </section>
      <template v-if="tickers.length">
        <hr class="w-full border-t border-gray-600 my-4" />
        <div>
          <button
            @click="page = page - 1"
            class="my-4 inline-flex items-center py-2 px-4 border border-transparent shadow-sm text-sm leading-4 font-medium rounded-full text-white bg-gray-600 hover:bg-gray-700 transition-colors duration-300 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-gray-500"
          >
            Previous
          </button>
          <button
            @click="page = page + 1"
            v-if="hasNextPage"
            class="my-4 mr-4 inline-flex items-center py-2 px-4 border border-transparent shadow-sm text-sm leading-4 font-medium rounded-full text-white bg-gray-600 hover:bg-gray-700 transition-colors duration-300 focus:outline-none focus:ring-2 focus:ring-offset-2 focus:ring-gray-500"
          >
            Next
          </button>
          <div class="flex gap-2">
            <label for="search">Filters:</label>
            <input id="search" v-model="filter" />
          </div>
        </div>
        <hr class="w-full border-t border-gray-600 my-4" />
        <dl class="mt-5 grid grid-cols-1 sm:grid-cols-2 w-full gap-5 md:grid-cols-3">
          <div
            @click="select(t)"
            v-for="t in filteredTickers()"
            :key="t.name"
            :class="{ 'border-purple-800': sel == t }"
            class="bg-white overflow-hidden w-full shadow rounded-lg border-4 border-solid cursor-pointer"
          >
            <div class="px-4 py-5 sm:p-6 text-center">
              <dt class="text-sm font-medium text-gray-500 truncate">{{ t.name }} - USD</dt>
              <dd class="mt-1 text-3xl font-semibold text-gray-900">{{ t.price }}</dd>
            </div>
            <div class="w-full border-t border-gray-200"></div>
            <button
              @click.stop="handleDelete(t)"
              class="flex items-center justify-center font-medium w-full bg-gray-100 px-4 py-4 sm:px-6 text-md text-gray-500 hover:text-gray-600 hover:bg-gray-200 hover:opacity-20 transition-all focus:outline-none"
            >
              <svg
                class="h-5 w-5"
                xmlns="http://www.w3.org/2000/svg"
                viewBox="0 0 20 20"
                fill="#718096"
                aria-hidden="true"
              >
                <path
                  fill-rule="evenodd"
                  d="M9 2a1 1 0 00-.894.553L7.382 4H4a1 1 0 000 2v10a2 2 0 002 2h8a2 2 0 002-2V6a1 1 0 100-2h-3.382l-.724-1.447A1 1 0 0011 2H9zM7 8a1 1 0 012 0v6a1 1 0 11-2 0V8zm5-1a1 1 0 00-1 1v6a1 1 0 102 0V8a1 1 0 00-1-1z"
                  clip-rule="evenodd"
                ></path></svg
              >Delete
            </button>
          </div>
        </dl>
        <hr class="w-full border-t border-gray-600 my-4" />
      </template>

      <section v-if="sel" class="relative">
        <h3 class="text-lg leading-6 font-medium text-gray-900 my-8">{{ sel?.name }} - USD</h3>
        <div class="flex items-end border-gray-600 border-b border-l h-64">
          <div
            v-for="(bar, idx) in normalizeGraph()"
            :key="idx"
            :style="{ height: `${bar}%` }"
            class="bg-purple-800 border w-10"
          ></div>
        </div>
        <button @click="sel = nul" type="button" class="absolute top-0 right-0">
          <svg
            xmlns="http://www.w3.org/2000/svg"
            xmlns:xlink="http://www.w3.org/1999/xlink"
            xmlns:svgjs="http://svgjs.com/svgjs"
            version="1.1"
            width="30"
            height="30"
            x="0"
            y="0"
            viewBox="0 0 511.76 511.76"
            style="enable-background: new 0 0 512 512"
            xml:space="preserve"
          >
            <g>
              <path
                d="M436.896,74.869c-99.84-99.819-262.208-99.819-362.048,0c-99.797,99.819-99.797,262.229,0,362.048    c49.92,49.899,115.477,74.837,181.035,74.837s131.093-24.939,181.013-74.837C536.715,337.099,536.715,174.688,436.896,74.869z     M361.461,331.317c8.341,8.341,8.341,21.824,0,30.165c-4.16,4.16-9.621,6.251-15.083,6.251c-5.461,0-10.923-2.091-15.083-6.251    l-75.413-75.435l-75.392,75.413c-4.181,4.16-9.643,6.251-15.083,6.251c-5.461,0-10.923-2.091-15.083-6.251    c-8.341-8.341-8.341-21.845,0-30.165l75.392-75.413l-75.413-75.413c-8.341-8.341-8.341-21.845,0-30.165    c8.32-8.341,21.824-8.341,30.165,0l75.413,75.413l75.413-75.413c8.341-8.341,21.824-8.341,30.165,0    c8.341,8.32,8.341,21.824,0,30.165l-75.413,75.413L361.461,331.317z"
                fill="#718096"
                data-original="#000000"
              ></path>
            </g>
          </svg>
        </button>
      </section>
    </div>
  </div>
</template>
