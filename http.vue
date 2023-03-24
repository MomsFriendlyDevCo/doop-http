<script lang="js" frontend>
import axios from 'axios';

/**
* Axios wrapper
* @url https://github.com/axios/axios#example
*
* @example Make a GET request and process the (array'd) document body)
* this.$http.get('/api/widgets').then(res => res.data)
*
* @example Make a complex request
* this.$http({
*   method: 'PUT',
*   headers: {'My Custom Header': 123},
*   params: {id: 123},
*   data: {someData: 'someValue'},
*/
// Make Axios request JSON by default
axios.defaults.headers.common.Accept = 'application/json';

// Make Axios encode using jQueries parameter serializer to keep Monoxide happy
axios.defaults.paramsSerializer = params =>
	$.param(params)
		.replace(/\bq=(.+)\b/g, p => p.replace(/%3A/g, ':')) // Allow `q=` to contain ':' tags

// Monkey patch Axios so that any error response gets correctly decoded rather than weird stuff like 'Server returned a 403 code'
axios.interceptors.response.use(response => response, error => {
	if (!error.response || !error.response.status) { // Recommended method to catch network errors as per https://github.com/axios/axios/issues/383#issuecomment-234079506
		return Promise.reject('Network error');
	} else if (error.response && error.response.data
		&& _.startsWith(error.response.request.responseURL, window.location.origin) // Only redirect hits to domain
		&& (error.response.status === 401 || error.response.status === 403)) {
		// FIXME: Some API routes may wish to throw 401 or 403 while actually being authenticated.
		app.router.go('/login');
		return Promise.reject(error.response.data);
	} else if (error.response && error.response.data) {
		return Promise.reject(error.response.data);
	} else {
		return Promise.reject(error.response);
	}
});

// Set BaseURL if we are running under cordova
app.ready.then(()=> {
	if (app.isCordova) {
		console.log('[$service.http]', 'Override baseUrl:', this.$config.apiUrl);
		axios.defaults.baseURL = this.$config.apiUrl;
	}
});


// Track active requests being made right now as axios.activeRequests {{{
axios.activeRequests = 0;

axios.interceptors.request.use(config => {
	axios.activeRequests++;
	return config;
});

axios.interceptors.response.use(
	res => {
		axios.activeRequests--;
		return res;
	},
	err => {
		axios.activeRequests--;
		return Promise.reject(err);
	},
);
// }}}

// Utility: $http.cleanUrl {{{
/**
* Clean a URL to remove all `undefined` values before requesting it
* @param {String} url The URL to clearn
* @param {Object} [options] Additional options to mutate behaviour
* @param {Array<String>} [options.remove] Array of values which should be considered 'empty' and should be removed. Defaults to falsy + 'null' + 'undefined'
* @returns {String} The input URL without any removeable params
*/
axios.cleanUrl = function cleanUrl(url, options) {
	let settings = {
		remove: ['', 'null', 'undefined'],
		...options,
	};

	let u = new URL(url, window.location.href);
	u.searchParams.forEach((v, k) => {
		if (settings.remove.includes(v))
			u.searchParams.delete(k);
	})

	return u.toString();
};
// }}}

// Utility: mongoQueryIn {{{
/**
* Utility function to return a ReST / Mongoosy+ReST compatible expression for a field
* This takes all the selected items, combines the IDs and returns either a single expression (if one item selected) or an `$in` expression if multiple
*
* @param {String} [field='status'] The name of the field in the output object
* @param {Object|Array} values The values to construct from, either an array of keys or an object (where only truthy keys are taken)
* @param {Object} [format='$in'] How to represent the data. ENUM: '$in' (return a Mongo '$in' query where appropriate), 'csv' (return a CSV of selected fields)
* @returns {Object|Null} Either a ReST compatible query or Null
*
* @example Compress all selected statuses suitable for a rest query
* axios.get('/api/widgets', {
*   params: {
*     ...this.$http.mongoQueryIn('status'),
*   },
* });
*/
axios.mongoQueryIn = function(field = 'status', values, format = '$in') {
	// Make a list of IDs that are selected
	let selected =
		Array.isArray(values) ? values
		: typeof values == 'object' ?
			Object.entries(values)
			.filter(([key, val]) => val)
			.map(([key]) => key)
		: (()=> { throw new Error('axios.mongoQueryIn() can only accept arrays or Objects') })();

	if (selected.length == 0) { // Nothing - return null
		return null;
	} else if (selected.length == 1) { // 1 Selected - Return simple {key: val}
		return {[field]: selected[0]};
	} else if (format == '$in') { // Multiple selected - use `{KEY: {$in: VALUES}}`
		return {[field]: {$in: selected}};
	} else if (format == 'csv') { // Multiple selected - use `{KEY: VAL1,VAL2...}`
		return {[field]: selected.join(',')};
	} else {
		throw new Error(`Unknown output format "${format}"`);
	}
};
// }}}

app.service('$http', ()=> axios); // NOTE: Because Axios is a function (with static methods) we have to wrap the return in an arrow function return to not confuse the app.service() scope fetcher
</script>
