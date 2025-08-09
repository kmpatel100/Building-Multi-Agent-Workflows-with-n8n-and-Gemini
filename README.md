# Prompt Chaining Multi-Agent Workflow with n8n and Google Gemini

💡 Hands-on resources for the Google Developer Club session on building intelligent multi-agent workflows using n8n and Gemini (Google AI). Includes sample workflows, API integrations, and setup guides.

---

## What You Will Build

1. Takes a blog topic from chat input  
2. Creates a blog outline  
3. Revises the outline  
4. Writes the full blog in Markdown  
5. Converts it to HTML  
6. Formats the email and sends it

---

## Prerequisites

- Windows, macOS, or Linux laptop  
- [Docker Desktop](https://www.docker.com/products/docker-desktop/)  
- [ngrok account](https://ngrok.com/) 
- [Google AI Studio](https://aistudio.google.com/) for a Gemini API key  
- Optional if you want the email step - a Gmail account connected to n8n via OAuth

---

## 1. Install Docker Desktop

Go to the [Docker Desktop site](https://www.docker.com/products/docker-desktop/) and download Docker Desktop for your operating system.

**Note for Windows:** Download **Windows - AMD64** unless you have an ARM based Snapdragon device.

---

## 2. Set Up ngrok

1. Create a free account at [ngrok](https://ngrok.com/)  
2. Install ngrok for your OS

Copy the **Forwarding URL** (for example `https://abc123.ngrok.app`) and use it in the environment variables below for `N8N_EDITOR_BASE_URL` and `WEBHOOK_URL`.

***Tip*** - get a static domain in ngrok if you want a stable URL.

---

## 3. Run n8n in Docker

You can also follow this detailed blog and YouTube video:  
👉 [How to Run n8n with ngrok Using Docker Desktop](https://blog.realiq.ca/p/how-to-run-n8n-with-ngrok-using-docker-desktop-b2b432362b794d5d)
👉 [Watch on YouTube – Self-Host n8n with Ngrok & Docker](https://www.youtube.com/watch?v=LoYBJ5djuOQ) 

### A) Pull the official image

1. Open Docker Desktop  
2. Go to **Images** and search for `n8nio/n8n`  
3. Click **Pull**

### B) Run the container with Optional Settings

- **Container Name:** `n8n_io`  
- **Ports:**  
  - Host Port: `5678`  
  - Container Port: `5678`  
- **Volumes:**  (Where the data is stored for n8n)
  - Host Path: `/home/your-username/n8n_data`  
    - Avoid OneDrive synced folders like Documents or Downloads  
  - Container Path: `/home/node/.n8n`  
- **Environment Variables:**

| Variable                     | Value                       |
|-----------------------------|-----------------------------|
| N8N_EDITOR_BASE_URL         | https://your-ngrok-domain   |
| WEBHOOK_URL                 | https://your-ngrok-domain   |
| N8N_DEFAULT_BINARY_DATA_MODE| filesystem                  |

Click **Run** when finished.

---

## 5. Credentials in n8n

### Google Gemini

- Go to **Credentials** in n8n  
- Create **Google Gemini (PaLM) API** credentials  
- Paste your API key from Google AI Studio  
- Save

### Gmail (optional, for sending the email)

- Go to **Credentials**  
- Create **Gmail OAuth2** credentials  
- Complete the OAuth flow  
- Save

---

## 7. Understand the Nodes and Flow

1. **When chat message received** - chat trigger  
2. **Outline Writer** - drafts the outline  
3. **Blog evaluator** - improves the outline  
4. **Blog Writer** - writes the blog in Markdown  
5. **Markdown to html** - converts Markdown to HTML  
6. **additional formatting for email** - wraps HTML for email  
7. **Send a message** - Gmail sends the result

### Models

- Outline Writer uses **Gemini 2.0 Flash**  
- Blog evaluator uses **Gemini 2.5 Flash**  
- Blog Writer uses **Gemini 2.5 Pro**

## 8. Exact Prompts from the Workflow

### Outline Writer agent

**User content**
```
Here is the topic to write a blog about: {{ $json.chatInput }}
```

**System message**
```
# Overview
You are an expert outline writer. Your job is to generate a structured outline for a blog post with section titles and key points.
```

---

### Blog evaluator agent

**User content**
```
Here is the outline:

{{ $json.output }}
```

**System message**
```
# Overview
You are an expert blog evaluator. Revise this outline and ensure it covers the following key criteria:
(1) Engaging Introduction
(2) Clear Section Breakdown
(3) Logical Flow
(4) Conclusion with Key Takeaways

## Output
Only output the revised outline.
```

---

### Blog Writer agent

**User content**
```
Here if the revised outline: {{ $json.output }}
```

**System message**
```
# Overview
You are an expert blog writer. Generate a detailed blog post using the outline with well-structured paragraphs and engaging content. Provide output in markdown format. Document structure should follow this guideline.

1. Title + Optional Subtitle
2. Author & Date
3. Intro
   - Briefly define the topic
   - Explain its importance
   - Mention what the post will cover
4. Table of Contents (main sections only)
5. Sections
   - Definition/Overview: Clear explanation of the concept
   - Practical Advice/Guidelines: Rules of thumb, tables, or bullet points
   - Comparison/Alternative: Pros and cons of related options
   - Step-by-Step Guide: Numbered instructions or code examples
6. Final Thoughts
   - Summary and personal recommendation
7. Call to Action (subscribe, comment, etc.)

Formatting Rules:
- Use markdown headings
- Keep language simple and clear
- Use bullet lists or tables for clarity
- Add code blocks where relevant
```

---

## 9. Code Nodes for Formatting

### Markdown to html - Function code
```js
return items.map(item => {
  const markdown = item.json.output;

  const html = markdown
    .replace(/^## (.*$)/gim, '<h2>$1</h2>')
    .replace(/^### (.*$)/gim, '<h3>$1</h3>')
    .replace(/^# (.*$)/gim, '<h1>$1</h1>')
    .replace(/\*\*(.*?)\*\*/gim, '<strong>$1</strong>')
    .replace(/\*(.*?)\*/gim, '<em>$1</em>')
    .replace(/\n/g, '<br />')
    .replace(/\[(.*?)\]\((.*?)\)/gim, '<a href="$2">$1</a>');

  return {
    json: {
      ...item.json,
      html
    }
  };
});
```

### additional formatting for email - Function code
```js
const htmlBlocks = items
  .map(item => item.json.html)
  .join('<hr style="margin:40px 0; border:0; border-top:1px solid #ccc;" />');

let finalHtml = `
<!DOCTYPE html>
<html style="margin:0; padding:0;">
  <head>
    <meta charset="UTF-8" />
    <style>
      html, body {
        margin: 0;
        padding: 0;
        font-family: Arial, sans-serif;
        background-color: #ffffff;
        color: #333333;
        line-height: 1.6;
      }
      .wrapper {
        max-width: 640px;
        margin: 0 auto;
        padding: 20px;
      }
      h1 {
        font-size: 28px;
        color: #111111;
        margin-bottom: 30px;
      }
      h2 {
        font-size: 20px;
        color: #0056b3;
        margin-top: 30px;
        margin-bottom: 10px;
      }
      a {
        color: #007acc;
        text-decoration: none;
      }
      a:hover {
        text-decoration: underline;
      }
      hr {
        border: none;
        border-top: 1px solid #ccc;
      }
    </style>
  </head>
  <body>
    <div class="wrapper">
      ${htmlBlocks}
    </div>
  </body>
</html>
`;

// Remove unwanted empty paragraph and leading whitespace
finalHtml = finalHtml
  .replace(/<p style="white-space:pre-line">\s*<\/p>/g, '')
  .replace(/^\s+/, '');

return [{ json: { html: finalHtml } }];
```

---

## 10. Gmail Node Settings

- **To:** `test0053@gmail.com` (change this to your email or disable the node during the demo)  
- **Subject:**  
  ```
  ={{ $('When chat message received').item.json.chatInput }}
  ```
- **Message:**  
  ```
  ={{ $json.html }}
  ```

Make sure the Gmail node uses your Gmail OAuth2 credentials.

---

## 11. How to Run the Demo
You can download the workflow file after the session is over.
1. Start Docker and confirm n8n is running at `http://localhost:5678`  
2. Start ngrok and copy the forwarding URL  
3. Confirm `N8N_EDITOR_BASE_URL` and `WEBHOOK_URL` are set to the ngrok URL  
4. Open the workflow in n8n  
5. Click **Execute Workflow**  
6. Open the **Chat** link from the trigger node  
7. Enter a topic, for example:  
   ```
   Best Productivity Hacks That Actually Work
   ```
8. Watch the chain:  
   - Outline generated  
   - Outline revised  
   - Full blog written  
   - Markdown converted to HTML  
   - HTML wrapped for email  
   - Email sent (if Gmail is configured)

---

## 12. Troubleshooting

- ngrok shows 502 or 403 - confirm n8n is running on port 5678 and the ngrok command targets `http 5678`  
- Chat UI does not appear - ensure the chat trigger is enabled and the workflow is active or running  
- Gemini API errors - recheck your Google Gemini credentials and model names  
- Gmail send fails - redo Gmail OAuth and select the credential in the node

---

## 13. Extend the Exercise

- Add a **Fact Checker** agent before the Blog Writer  
- Add an **SEO Optimizer** agent to generate title, meta description, and keywords  
- Replace Gmail with Google Docs, Notion, Slack, or an HTTP Request to publish the HTML  
- Add a **Translation** agent to output multiple languages  
- Swap models to compare speed and quality between 2.0 Flash, 2.5 Flash, and 2.5 Pro

---
## Next Steps

Congratulations, you have just built your first **prompt chaining multi-agent workflow** with **n8n** and **Google Gemini**.

From here, you can:

- Experiment with different prompts for each agent
- Swap models between speed-optimized and quality-optimized ones
- Add new agents to extend functionality
- Connect the output to more tools (Docs, Notion, Slack, APIs)

Keep exploring, the skills you learned today can be applied to automate reports, research tasks, content creation, and much more.

Happy automating! 🚀
