# test-
import java.io.*;
import java.net.*;
import java.util.*;
import javax.xml.parsers.*;
import org.w3c.dom.*;

public class PubMedFetcher {

    public static List<Map<String, String>> fetchPapers(String query) throws Exception {
        String baseUrl = "https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi";
        String params = "?db=pubmed&term=" + URLEncoder.encode(query, "UTF-8") + "&retmode=xml&retmax=10";

        URL url = new URL(baseUrl + params);
        Document doc = getXMLDocument(url);

        List<String> ids = new ArrayList<>();
        NodeList idNodes = doc.getElementsByTagName("Id");
        for (int i = 0; i < idNodes.getLength(); i++) {
            ids.add(idNodes.item(i).getTextContent());
        }

        String detailsUrl = "https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esummary.fcgi";
        String detailsParams = "?db=pubmed&id=" + String.join(",", ids) + "&retmode=xml";
        URL detailsUrlFull = new URL(detailsUrl + detailsParams);
        Document detailsDoc = getXMLDocument(detailsUrlFull);

        List<Map<String, String>> papers = new ArrayList<>();
        NodeList docSums = detailsDoc.getElementsByTagName("DocSum");
        for (int i = 0; i < docSums.getLength(); i++) {
            Element docSum = (Element) docSums.item(i);
            Map<String, String> paper = new HashMap<>();
            paper.put("PubmedID", getElementText(docSum, "Id"));
            paper.put("Title", getItemText(docSum, "Title"));
            paper.put("Publication Date", getItemText(docSum, "PubDate"));
            paper.put("Non-academic Author(s)", "N/A");
            paper.put("Company Affiliation(s)", "N/A");
            paper.put("Corresponding Author Email", "N/A");
            papers.add(paper);
        }

        return papers;
    }

    private static Document getXMLDocument(URL url) throws Exception {
        DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
        DocumentBuilder builder = factory.newDocumentBuilder();
        return builder.parse(url.openStream());
    }

    private static String getElementText(Element element, String tagName) {
        NodeList nodes = element.getElementsByTagName(tagName);
        return (nodes.getLength() > 0) ? nodes.item(0).getTextContent() : "N/A";
    }

    private static String getItemText(Element element, String itemName) {
        NodeList items = element.getElementsByTagName("Item");
        for (int i = 0; i < items.getLength(); i++) {
            Element item = (Element) items.item(i);
            if (item.getAttribute("Name").equals(itemName)) {
                return item.getTextContent();
            }
        }
        return "N/A";
    }

    public static void saveToCSV(List<Map<String, String>> papers, String filename) throws IOException {
        try (BufferedWriter writer = new BufferedWriter(new FileWriter(filename))) {
            String[] headers = {"PubmedID", "Title", "Publication Date", "Non-academic Author(s)", "Company Affiliation(s)", "Corresponding Author Email"};
            writer.write(String.join(",", headers) + "\n");

            for (Map<String, String> paper : papers) {
                List<String> values = new ArrayList<>();
                for (String header : headers) {
                    values.add(paper.getOrDefault(header, "N/A"));
                }
                writer.write(String.join(",", values) + "\n");
            }
        }
    }

    public static void main(String[] args) throws Exception {
        if (args.length == 0) {
            System.out.println("Usage: java PubMedFetcher <query> [-f <filename>] [-d]");
            return;
        }

        String query = args[0];
        boolean debug = Arrays.asList(args).contains("-d");
        String filename = Arrays.asList(args).contains("-f") ? args[Arrays.asList(args).indexOf("-f") + 1] : null;

        if (debug) {
            System.out.println("Fetching papers for query: " + query);
        }

        List<Map<String, String>> papers = fetch_papers(query);

        if (filename != null) {
            saveToCSV(papers, filename);
            System.out.println("Results saved to " + filename);
        } else {
            for (Map<String, String> paper : papers) {
                System.out.println(paper);
            }
        }
