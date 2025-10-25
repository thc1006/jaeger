# Elasticsearch Storage Layer Map

## Overview
This document maps the public types, interfaces, and main methods in the Elasticsearch storage layer.

## Directory Structure
```
internal/storage/elasticsearch/client.go
internal/storage/elasticsearch/client/basic_auth.go
internal/storage/elasticsearch/client/client.go
internal/storage/elasticsearch/client/cluster_client.go
internal/storage/elasticsearch/client/ilm_client.go
internal/storage/elasticsearch/client/index_client.go
internal/storage/elasticsearch/client/interfaces.go
internal/storage/elasticsearch/client/mocks/mocks.go
internal/storage/elasticsearch/config/auth_helper.go
internal/storage/elasticsearch/config/config.go
internal/storage/elasticsearch/dbmodel/dot_replacer.go
internal/storage/elasticsearch/dbmodel/model.go
internal/storage/elasticsearch/errors.go
internal/storage/elasticsearch/filter/alias.go
internal/storage/elasticsearch/filter/date.go
internal/storage/elasticsearch/mocks/mocks.go
internal/storage/elasticsearch/query/range_query.go
internal/storage/elasticsearch/textTemplate.go
internal/storage/elasticsearch/wrapper/wrapper.go
internal/storage/elasticsearch/wrapper/wrapper_nolint.go
```

## Key Interfaces

### client/interfaces.go
type IndexAPI interface {
	GetJaegerIndices(prefix string) ([]Index, error)
	IndexExists(index string) (bool, error)
	AliasExists(alias string) (bool, error)
	DeleteIndices(indices []Index) error
	CreateIndex(index string) error
	CreateAlias(aliases []Alias) error
	DeleteAlias(aliases []Alias) error
	CreateTemplate(template, name string) error
	Rollover(rolloverTarget string, conditions map[string]any) error
}

type ClusterAPI interface {
	Version() (uint, error)
}

type IndexManagementLifecycleAPI interface {
	Exists(name string) (bool, error)
}

## Core Client Types

### client/client.go
type ResponseError struct {
type Client struct {
type elasticRequest struct {
func (c *Client) request(esRequest elasticRequest) ([]byte, error) {
func (c *Client) setAuthorization(r *http.Request) {
func (*Client) handleFailedRequest(res *http.Response) error {

## Configuration

### config/config.go
type IndexOptions struct {
type Indices struct {
type bulkCallback struct {
func (p IndexPrefix) Apply(indexName string) string {
type Configuration struct {
type TagsAsFields struct {
type Sniffing struct {
type BulkProcessing struct {
type TokenAuthentication struct {
type Authentication struct {
type BasicAuthentication struct {
func NewClient(ctx context.Context, c *Configuration, logger *zap.Logger, metricsFactory metrics.Factory) (es.Client, error) {
func (bcb *bulkCallback) invoke(id int64, requests []elastic.BulkableRequest, response *elastic.BulkResponse, err error) {
func newElasticsearchV8(ctx context.Context, c *Configuration, logger *zap.Logger) (*esv8.Client, error) {
func setDefaultIndexOptions(target, source *IndexOptions) {
func (c *Configuration) ApplyDefaults(source *Configuration) {
func RolloverFrequencyAsNegativeDuration(frequency string) time.Duration {
func (c *Configuration) TagKeysAsFields() ([]string, error) {
func (c *Configuration) getESOptions(disableHealthCheck bool) []elastic.ClientOptionFunc {
func (c *Configuration) getConfigOptions(ctx context.Context, logger *zap.Logger) ([]elastic.ClientOptionFunc, error) {
func addLoggerOptions(options []elastic.ClientOptionFunc, logLevel string, logger *zap.Logger) ([]elastic.ClientOptionFunc, error) {
func GetHTTPRoundTripper(ctx context.Context, c *Configuration, logger *zap.Logger) (http.RoundTripper, error) {
func loadTokenFromFile(path string) (string, error) {
func (c *Configuration) Validate() error {

## Error Handling

### errors.go
func DetailedError(err error) error {

## Database Model

### dbmodel/model.go
type Trace struct {
type Span struct {
type Reference struct {
type Process struct {
type Log struct {
type KeyValue struct {
type Service struct {
type Operation struct {
type OperationQueryParameters struct {
type TraceQueryParameters struct {

## Wrapper Layer

### wrapper/wrapper.go
type ClientWrapper struct {
func (c ClientWrapper) GetVersion() uint {
func WrapESClient(client *elastic.Client, s *elastic.BulkProcessor, esVersion uint, clientV8 *esv8.Client) ClientWrapper {
func (c ClientWrapper) IndexExists(index string) es.IndicesExistsService {
func (c ClientWrapper) CreateIndex(index string) es.IndicesCreateService {
func (c ClientWrapper) DeleteIndex(index string) es.IndicesDeleteService {
func (c ClientWrapper) CreateTemplate(ttype string) es.TemplateCreateService {
func (c ClientWrapper) Index() es.IndexService {
func (c ClientWrapper) Search(indices ...string) es.SearchService {
func (c ClientWrapper) MultiSearch() es.MultiSearchService {
func (c ClientWrapper) Close() error {
type IndicesExistsServiceWrapper struct {
func WrapESIndicesExistsService(indicesExistsService *elastic.IndicesExistsService) IndicesExistsServiceWrapper {
func (e IndicesExistsServiceWrapper) Do(ctx context.Context) (bool, error) {
type IndicesCreateServiceWrapper struct {
func WrapESIndicesCreateService(indicesCreateService *elastic.IndicesCreateService) IndicesCreateServiceWrapper {
func (c IndicesCreateServiceWrapper) Body(mapping string) es.IndicesCreateService {
func (c IndicesCreateServiceWrapper) Do(ctx context.Context) (*elastic.IndicesCreateResult, error) {
type TemplateCreateServiceWrapper struct {
type IndicesDeleteServiceWrapper struct {
func WrapESIndicesDeleteService(indicesDeleteService *elastic.IndicesDeleteService) IndicesDeleteServiceWrapper {
func (e IndicesDeleteServiceWrapper) Do(ctx context.Context) (*elastic.IndicesDeleteResponse, error) {
func WrapESTemplateCreateService(mappingCreateService *elastic.IndicesPutTemplateService) TemplateCreateServiceWrapper {
func (c TemplateCreateServiceWrapper) Body(mapping string) es.TemplateCreateService {
func (c TemplateCreateServiceWrapper) Do(ctx context.Context) (*elastic.IndicesPutTemplateResponse, error) {
type TemplateCreatorWrapperV8 struct {
func (c TemplateCreatorWrapperV8) Body(mapping string) es.TemplateCreateService {
func (c TemplateCreatorWrapperV8) Do(context.Context) (*elastic.IndicesPutTemplateResponse, error) {
type IndexServiceWrapper struct {
func WrapESIndexService(indexService *elastic.BulkIndexRequest, bulkService *elastic.BulkProcessor, esVersion uint) IndexServiceWrapper {

## Main Client Entry Point

### client.go
type Client interface {
type IndicesExistsService interface {
type IndicesCreateService interface {
type IndicesDeleteService interface {
type TemplateCreateService interface {
type IndexService interface {
type SearchService interface {
type MultiSearchService interface {

---
Generated: Sat Oct 25 18:04:20 UTC 2025
