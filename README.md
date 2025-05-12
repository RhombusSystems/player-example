# Rhombus Systems Camera Player Example

This repository provides a simple example of how to integrate and stream video from Rhombus security cameras in your web application using DashJS.

## Overview

The Rhombus Camera Player Example demonstrates how to:
- Authenticate with Rhombus API using federated session tokens
- Retrieve media streaming URIs for specific cameras
- Set up a video player with DashJS to stream camera feeds
- Configure optimized streaming settings for security camera footage

## Prerequisites

- Rhombus Systems account with API access
- Camera UUID from your Rhombus dashboard
- A server to proxy API requests to Rhombus (for security reasonse)

## Quick Start

1. Clone this repository
2. Edit `index.html` and set the following variables:
   - `CAMERA_UUID`: The UUID of your Rhombus camera
   - `BASE_URL`: Your server's base URL (e.g., "https://www.example.com/api")
   - `GET_FEDERATED_TOKEN_PATH`: Your server's path to proxy the federated token API
   - `GET_MEDIA_URIS_PATH`: Your server's path to proxy the media URIs API

## Server Setup

You'll need to set up a server that acts as a proxy to the Rhombus API. Your server should:

1. Handle authentication with the Rhombus API using your API token
2. Expose endpoints that forward requests to:
   - `/org/generateFederatedSessionToken` (to get a federated token)
   - `/camera/getMediaUris` (to get media streaming URLs)

This proxy approach protects your API tokens and adds a layer of security.

## How It Works

The example follows these steps:
1. Obtains a federated session token from Rhombus via your proxy server
2. Uses the token to request media URIs for the specified camera
3. Initializes a DashJS player with the media URI
4. Configures the player with optimized settings for security footage
5. Begins streaming the video feed

## Customization

The example includes a comprehensive set of DashJS settings that are optimized for security camera feeds. You can adjust these settings to:
- Control buffer size and behavior
- Modify live catch-up behavior
- Adjust quality settings
- Fine-tune streaming performance

## Troubleshooting

If you encounter issues:
- Ensure your server correctly proxies requests to the Rhombus API
- Verify your camera UUID is correct
- Check browser console for JavaScript errors
- Ensure your Rhombus account has proper permissions

## Security Considerations

- Never expose your Rhombus API tokens directly in frontend code
- Use appropriate access controls for your proxy server
- Consider implementing token validation and request sanitization
- Use HTTPS for all API requests

## Additional Resources
- For any questions or support, please contact us at [api@rhombus.com](mailto:api@rhombus.com)
- [DashJS Documentation](https://github.com/Dash-Industry-Forum/dash.js/wiki)
- [Rhombus Systems API Documentation](https://docs.rhombussystems.com/reference)