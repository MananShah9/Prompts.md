Hey team,

Just putting a thought forward regarding keepie.

Instead of sending authorization tokens to the receipt URL, what if we send the required data directly??

This would eliminate the need for tokens in the first place (less headache on the server).

For example, if DMDLens wants month-wise release frequency from qlik-api, instead of DMDLens receiving an auth token from keepie, it could directly receive the release frequency data at the receipt URL.

In short, Instead of making request for token just make request with requirement and receipt url...

I understand that this is only suitable for apps that exclusively serve GET requests. There are systems (I know Qlik DMD api) that do have strictly get endpoints...

Would really like to discuss and hear opinions on this!


And we could name the architecture SKIPie 😂