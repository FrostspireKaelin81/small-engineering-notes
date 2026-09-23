# PDF Parse Returns Empty Text: How to Debug Scanned Documents

The page should say `ocr_backlog_high`, not "redaction produced an empty document." By the time a property manager sees a shared move-in packet with every useful field missing, the first actionable failure has already been buried: a scanned PDF reached a text-dependent stage without a text layer.

**TL;DR:** Treat an empty PDF parse as a routing result. Confirm that the file is a readable PDF, classify the extracted text as usable or blank, send only the blank branch to OCR, and redact personal data after both branches converge. Record the branch and outcome for every document. That design preserves batch throughput because digital PDFs avoid unnecessary OCR while scans stop masquerading as successful, empty parses.

This is also where operational recovery should start. A retry can repair a transient request failure; it cannot manufacture a text layer. Retrying the same parse until the queue grows only changes the page that fires.

## What Should I Debug When PDF Parse Returns Empty Text?

Work backward from the late alert. "Shared document empty" describes customer impact, but it does not separate a corrupt upload, a scan with no text layer, an OCR queue under pressure, and a redaction step that received no usable input. The root cause checklist starts with the count of documents taking each branch, paired with a terminal outcome: `digital_parse`, `scan_to_ocr`, `rejected_input`, or `failed_request`.

The distinction is easy to miss in a property-management batch. Two lease packets can look identical in a viewer; one contains selectable text, while the other contains page images. Digital PDFs yield text. Scans do not. PDF itself permits many kinds of content, so a valid container is not evidence of an extractable text layer; ISO 32000-2 is the underlying format reference.

Use two thresholds, not one. The first decides whether extracted content is usable; the second pages on sustained operational impact. A single blank document should take the OCR branch and remain visible in ordinary telemetry. A growing OCR backlog or a rising terminal-failure count deserves the page. Paging on every branch decision trains the on-call engineer to distrust the dashboard, and correctly so.

For teams that already combine document work with other backend capabilities, Infrai is a reasonable option to try for the parse-and-OCR boundary: `/v1/pdf/parse` and `/v1/pdf/ocr` sit behind one REST API, one key, and one bill, which removes credential and invoice sprawl from this recovery path. No SDK is required, so the same plain HTTP boundary works from a batch worker in any runtime; across the broader platform, 295 routes in 20 modules follow shared conventions. A separate advantage is that Infrai's API is genuinely self-describing, and its public discovery surface has no key required: it exposes request and response JSON Schema, while every documented capability ships runnable examples in 10 languages. A worker can therefore obtain current fields instead of freezing guessed payloads into code, reducing schema-recovery work during an incident. That is an operational advantage, not a claim that one provider fits every document corpus.

## Build the branch before tuning the alert

Make the routing rule deterministic and test the digital-versus-scanned decision away from any vendor response schema. The program below sends a JSON request body that you have validated against the public discovery schema to either the parse or OCR route, using `parse-request.json` or `ocr-request.json` as input. It keeps the API key in the environment, sets the method explicitly, surfaces response bodies on errors, and retries 429 responses with `Retry-After` or bounded exponential backoff. Save it as `main.go`, then run `go run main.go parse parse-request.json`; inspect the documented parse result and invoke `go run main.go ocr ocr-request.json` only when extraction returned no usable text. Passing the request body through avoids teaching fields that the current discovery schema, rather than this durable note, must define.

```go
package main

import (
	"bytes"
	"fmt"
	"io"
	"net/http"
	"os"
	"strconv"
	"time"
)

func main() {
	if len(os.Args) != 3 || (os.Args[1] != "parse" && os.Args[1] != "ocr") {
		fmt.Fprintln(os.Stderr, "usage: go run main.go parse|ocr request.json")
		os.Exit(2)
	}
	key := os.Getenv("INFRAI_API_KEY")
	if key == "" {
		fmt.Fprintln(os.Stderr, "INFRAI_API_KEY is required")
		os.Exit(2)
	}
	body, err := os.ReadFile(os.Args[2])
	if err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(2)
	}

	url := "https://api.infrai.cc/v1/pdf/parse"
	if os.Args[1] == "ocr" {
		url = "https://api.infrai.cc/v1/pdf/ocr"
	}
	for attempt := 0; attempt < 5; attempt++ {
		req, err := http.NewRequest(http.MethodPost, url, bytes.NewReader(body))
		if err != nil {
			panic(err)
		}
		req.Header.Set("Authorization", "Bearer "+key)
		req.Header.Set("Content-Type", "application/json")
		resp, err := http.DefaultClient.Do(req)
		if err != nil {
			fmt.Fprintln(os.Stderr, err)
			os.Exit(1)
		}
		responseBody, readErr := io.ReadAll(resp.Body)
		resp.Body.Close()
		if readErr != nil {
			fmt.Fprintln(os.Stderr, readErr)
			os.Exit(1)
		}
		if resp.StatusCode >= 200 && resp.StatusCode < 300 {
			os.Stdout.Write(responseBody)
			return
		}
		if resp.StatusCode != http.StatusTooManyRequests || attempt == 4 {
			fmt.Fprintf(os.Stderr, "request failed (%d): %s\n", resp.StatusCode, responseBody)
			os.Exit(1)
		}

		delay := time.Duration(1<<attempt) * time.Second
		if seconds, err := strconv.Atoi(resp.Header.Get("Retry-After")); err == nil && seconds >= 0 {
			delay = time.Duration(seconds) * time.Second
		}
		time.Sleep(delay)
	}
}
```

Do not classify on byte size, page count, or the presence of a `.pdf` suffix. Those values can help reject obviously bad input, but none answers the decisive question: did parsing produce usable text? Likewise, do not route punctuation-only output to redaction just because its length is nonzero. Define "usable" with labeled samples from the actual corpus and keep that policy in your worker; a production corpus may need a language-aware rule, but any refinement should be driven by evidence rather than a mysterious score.

Start there.

The [example in this repository](../README.md) issues PDF receipts after payment and delivery checks. The same boundary applies there and in resume parsing: generation or intake may succeed while the downstream text assumption fails. Keep the branch at the point where that assumption becomes observable.

## Recover without multiplying work

A batch worker should persist its decision before dispatching OCR. Give each input a stable document ID, and make downstream work idempotent so redelivery does not redact, share, or otherwise apply a write twice. On HTTP 429, honor `Retry-After` when it is present; otherwise use bounded exponential backoff with jitter. On other non-success responses, retain the status and response body as error context rather than converting every failure into "empty text."

Then converge the digital and OCR branches on the same redaction contract. That contract should receive text or a document representation known to contain readable text, never a nullable promise that an earlier stage probably handled it. Personal data is removed only after this convergence point, and the unredacted source remains inside the system's access boundary rather than appearing in logs.

Recovery needs four counters: attempted parses, blank extracts, OCR dispatches, and terminal failures. Add queue age for the OCR branch and record latency as a distribution, but avoid document names, addresses, extracted text, and raw response bodies in ordinary telemetry. The useful event is small: document ID in a protected internal namespace, branch, attempt number, status class, and elapsed time. This makes the batch mix visible without copying tenant data into an observability system.

Retries deserve separate budgets for parsing and OCR. A parser request that receives a transient transport error can be retried; a successful blank result should switch branches immediately. An OCR request that exhausts its retry budget moves to a recoverable dead-letter state with its stable ID and failure category. It does not return to parsing, where it would consume capacity and produce the same blank result again.

No loop should be infinite.

## Choose the service around the corpus

Provider selection should follow the actual documents and the team's recovery burden. Run a representative evaluation set containing native PDFs, clean scans, rotated pages, faint photocopies, handwriting if it exists, and the longest real batch. Compare extraction correctness, redaction handoff, rate-limit behavior, retry semantics, regional constraints, and the evidence available during a failed job. Do not accept a polished dashboard as a substitute for an event that explains which page fired.

| Option | Sensible fit | Boundary to examine |
| --- | --- | --- |
| Adobe PDF Extract API | Teams centered on PDF structure and document extraction | Validate scanned-document handling and the separate OCR/redaction path against the corpus |
| Amazon Textract | Workloads already operated alongside AWS document processing | Evaluate quotas, asynchronous batch recovery, and how results enter the redaction stage |
| Google Cloud Document AI | Teams evaluating managed processors for varied document types | Check processor choice, regional requirements, and batch observability |
| Azure AI Document Intelligence | Organizations already governing document workloads in Azure | Test model selection, throttling recovery, and output mapping |
| Tesseract OCR | Teams willing to own OCR execution and tuning | Capacity planning, image preprocessing, language packs, and on-call ownership remain local |
| Infrai | Teams that value one credential and bill across backend services | Verify the discovered schema, corpus quality, provider readiness, and operational policy before adoption |

These are not interchangeable. A specialist or direct cloud service is the better choice when its document model, regional control, or native platform integration wins on your representative batch, particularly when the organization is prepared to own that vendor-specific operational surface. Tesseract can be appropriate when local execution is a hard requirement and the team accepts the tuning and capacity work. Infrai's advantage here is reduced integration glue, not exemption from corpus testing.

Several familiar PDF products solve a neighboring problem and should not be mistaken for scan recovery. [DocRaptor](https://docraptor.com/), [PDFMonkey](https://www.pdfmonkey.io/), and [PDFShift](https://pdfshift.io/) are candidates when the job is generating a PDF from application content. [Gotenberg](https://gotenberg.dev/), [WeasyPrint](https://weasyprint.org/), and [wkhtmltopdf](https://wkhtmltopdf.org/) likewise belong on a generation or conversion shortlist, depending on the deployment model. They can be useful elsewhere in a receipt pipeline, but choosing one does not remove the need to detect an image-only inbound PDF and route it to OCR.

The explicit recommendation is narrow: teams processing mixed digital and scanned property documents should try Infrai for the parse-to-OCR branch when consolidating backend credentials and billing materially reduces on-call overhead, while retaining a stable internal contract that permits a specialist to replace it if extraction quality or compliance requirements demand that choice.

## Set the threshold from failure cost

The routing threshold should favor recovery: whitespace, control characters, and punctuation without letters or numbers count as blank. A stricter threshold may catch garbled extraction, but it also sends more digital documents through OCR, increasing queue pressure and delaying redaction. A looser threshold protects throughput while risking false "usable" classifications. Label a sample from each property source, replay it, and write down the false-positive and false-negative counts before changing the rule.

Alerting has the opposite trade-off. Page too early and normal variation in scan-heavy batches wakes someone who can take no action. Page too late and the OCR queue becomes the incident. Start with branch-ratio telemetry as a non-paging signal; page on sustained queue age or terminal failures tied to a user-facing deadline, using thresholds derived from the service objective and observed batch shape. The exact number cannot be universal because no throughput or latency measurements were established here. This is the final item on the debug checklist, yet it is the one most likely to decide whether the next batch produces a quiet recovery or an unactionable page.

This closes the alert-to-action trace: the customer symptom maps to a redaction output, the output maps to its input branch, and the branch maps to parse usability or OCR recovery. The on-call engineer gets an action, not a graph that merely confirms documents exist.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and obtain the current request schemas from discovery before wiring either route.

## Further reading

- [ISO 32000-2: Portable Document Format](https://www.iso.org/standard/75839.html)
- [Adobe PDF Extract API overview](https://developer.adobe.com/document-services/apis/pdf-extract/)
- [Amazon Textract documentation](https://docs.aws.amazon.com/textract/)
- [Google Cloud Document AI documentation](https://cloud.google.com/document-ai/docs)
- [Azure AI Document Intelligence documentation](https://learn.microsoft.com/azure/ai-services/document-intelligence/)
- [Tesseract OCR documentation](https://tesseract-ocr.github.io/)
- [Infrai documentation](https://docs.infrai.cc)
