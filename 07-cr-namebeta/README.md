### Code review of command line domain query tool

Repository: https://github.com/TimothyYe/namebeta
```
go get github.com/TimothyYe/namebeta
```

#### Common package review

 - Add `1.10` Go version to travis.

#### How it works

```
go build -o namebeta
./namebeta
./namebeta alex.com
```

#### main.go

Move this code to utils.go:

```
if len(os.Args) == 1 {
	displayUsage()
	os.Exit(0)
}
```

Can be combined into:

```
func main() {
	query(parseArgs(os.Args))
}
```

Also these 3 vars can be wrapped into a struct, so later we can easily change struct and don't change code which is handling return values.

It's not clear to do Exit and return values in one func. so we will reutn *cli and check it in main and print there.

#### utils.go

```
type cli struct {
	Domain   string
	WithMore bool
	GetWhois bool
}

func parseArgs(args []string) *cli {
	if len(os.Args) == 1 {
		return nil
	}

	switch args[1] {
	case more:
		if len(args) > 2 {
			return &cli{args[2], true, false}
		}
	case whois:
		if len(args) > 2 {
			return &cli{args[2], false, true}
		}
	default:
		return &cli{args[1], false, false}
	}

	return nil
}
```

Combine whoisQuery and domainQuery.

```
func getQueryResults(endpoint string, domain string, param map[string]string) []interface{} {
	var result []interface{}

	request := gorequest.New()
	_, body, _ := request.Post(endpoint).
		Type("form").
		Set("User-Agent", userAgent).
		Set("Refer", fmt.Sprintf(referURL, domain)).
		SendMap(param).End()

	if err := json.Unmarshal([]byte(body), &result); err != nil {
		color.Red(fmt.Sprintf("%s gailed to query %s endpoint. domain: %s \r\n", crossSymbol, endpoint, domain))
		os.Exit(1)
	}

	return result
}
```

Build Run!

We shouldn't skip errors!

Same as in parseArgs, we should return error.

```
func getQueryResults(endpoint string, domain string, param map[string]string) ([]interface{}, error) {
	var result []interface{}

	request := gorequest.New()
	_, body, errors := request.Post(endpoint).
		Type("form").
		Set("User-Agent", userAgent).
		Set("Refer", fmt.Sprintf(referURL, domain)).
		SendMap(param).End()

	if len(errors) > 0 {
		return nil, fmt.Errorf("failed to query %s endpoint. domain: %s, error: %v", endpoint, domain, errors[0])
	}

	if err := json.Unmarshal([]byte(body), &result); err != nil {
		return nil, fmt.Errorf("failed to query %s endpoint. domain: %s", endpoint, domain)
	}

	return result, nil
}
```

And return error in query:

#### query.go

```
func query(c *cli) error {
	if c.GetWhois {
		return queryWhois(c.Domain)
	}

	return queryDomain(c.Domain, c.WithMore)
}
```

#### main.go

```
err := query(cliVars)
if err != nil {
	color.Red(fmt.Sprintf("%s %s\r\n", crossSymbol, err.Error()))
	os.Exit(1)
}
```

So now we got rid from printing error in multiple places.

#### query.go

Move to withMore:

`if len(resultMore) > 0 && resultMore[0].(bool) {`

#### utils.go

One more thing I don't like is that we use gorequest package to get data from URL, while it can be easily done with net/http std package.

And now we can remove this package from godep.

```
godep restore
godep save
```

Build Run!

Yeah! we reduced dependencies!

#### query.go

I think we should check result slice length to be sure, otherwise we can get panic. It will be perfect to define response type via structs, but I checked that response is not well structured, it contains different types in one slice, so it will be messy to work with it, let's keep as is.

#### Results

```
git cm "review 1"
[master 2e7767e] review 1
28 files changed, 91 insertions(+), 12722 deletions(-)
```

Not bad improvement I think.