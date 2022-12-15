this.$http({
url: this.$http.adornUrl(''),
method: 'post',
data: this.$http.adornData(data, false)
}).then(({ data }) => { });